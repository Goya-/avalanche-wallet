<template>
    <div class="transfer_card">
        <h1>{{$t('transfer.title')}}</h1>
        <div v-if="networkStatus !== 'connected'" class="disconnected">
            <p>Unable to send assets. Disconnected from the network.</p>
        </div>
        <div class="card_body" v-else>
            <div class="new_order_Form">
                <div class="lists" v-show="!isConfirm">
                    <tx-list class="tx_list" ref="txList" @change="updateTxList"></tx-list>
                    <template v-if="hasNFT">
                        <NftList @change="updateNftList" ref="nftList"></NftList>
                    </template>
                </div>
                <div v-show="isConfirm" class="lists">
                    <TxSummary  class="lists" :orders="formOrders" :nft-orders="formNftOrders"></TxSummary>
                </div>
                <div>
                    <div class="to_address" >
                        <h4>{{$t('transfer.to')}}</h4>
                        <qr-input v-if="!isConfirm" v-model="addressIn" class="qrIn" placeholder="xxx"></qr-input>
                        <p class="confirm_val" v-else>{{formAddress}}</p>

                        <template v-if="isConfirm && formMemo.length>0">
                            <h4>Memo (Optional)</h4>
                            <p class="confirm_val">{{formMemo}}</p>
                        </template>
                        <template v-else-if="!isConfirm">
                            <h4>Memo (Optional)</h4>
                            <textarea class="memo" maxlength="256" placeholder="Memo" v-model="memo"></textarea>
                        </template>

                    </div>
                    <div class="fees">
                        <h4>{{$t('transfer.fees')}}</h4>
                        <p>{{$t('transfer.fee_tx')}} <span>{{txFee.toLocaleString(9)}} AVAX</span></p>
                    </div>
<!--                    <div class="advanced">-->
<!--                        <v-expansion-panels accordion class="advanced_panel" flat>-->
<!--                            <v-expansion-panel>-->
<!--                                <v-expansion-panel-header>{{$t('transfer.advanced')}}</v-expansion-panel-header>-->
<!--                                <v-expansion-panel-content>-->
<!--                                    <label>{{$t('transfer.adv_change')}}</label>-->
<!--                                    <p class="explain">Where to send the remaining assets after the transaction.</p>-->
<!--                                    <radio-buttons class="radio_buttons" :default_val="selectedAddress" :value="addresses" @change="changeAddressesChange"></radio-buttons>-->
<!--                                    &lt;!&ndash;                                <address-dropdown :default_val="selectedAddress" @change="changeAddressesChange"></address-dropdown>&ndash;&gt;-->
<!--                                </v-expansion-panel-content>-->
<!--                            </v-expansion-panel>-->
<!--                        </v-expansion-panels>-->
<!--                    </div>-->


                    <div class="checkout">
                        <ul class="err_list" v-if="formErrors.length>0">
                            <li v-for="err in formErrors" :key="err">{{err}}</li>
                        </ul>
                        <template v-if="!isConfirm">
                            <v-btn depressed class="button_primary" color="#4C2E56" :ripple="false" @click="confirm" :disabled="!canSend" block>Confirm</v-btn>
                        </template>
                        <template v-else-if="isConfirm && !isSuccess">
                            <p class="err">{{err}}</p>
                            <v-btn depressed class="button_primary" color="#4C2E56" :loading="isAjax" :ripple="false" @click="submit" :disabled="!canSend" block>{{$t('transfer.send')}}</v-btn>
                            <v-btn text block small style="margin-top: 20px !important; color: var(--primary-color);" @click="cancelConfirm">Cancel</v-btn>
                        </template>
                        <template v-else-if="isSuccess">
                            <p style="color: var(--success);"> <fa icon="check-circle"></fa> Transaction Sent </p>
                            <label style="word-break: break-all;"><b>ID: </b> {{txId}}</label>
                            <v-btn depressed style="margin-top: 14px;" class="button_primary" color="#4C2E56" :ripple="false" @click="startAgain" block>Start Again</v-btn>
                        </template>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
<script lang="ts">
    import 'reflect-metadata';
    import { Vue, Component, Ref } from 'vue-property-decorator';

    import TxList from "@/components/wallet/transfer/TxList.vue";
    import RadioButtons from "@/components/misc/RadioButtons.vue";
    import Big from "big.js";

    import NftList from "@/components/wallet/transfer/NftList.vue";

    //@ts-ignore
    import { QrInput } from "@avalabs/vue_components";
    import {ava, avm, isValidAddress} from "../../AVA";
    import FaucetLink from "@/components/misc/FaucetLink.vue";
    import {ITransaction} from "@/components/wallet/transfer/types";
    import { UTXO } from "avalanche/dist/apis/avm";
    import {Buffer, BN} from "avalanche";
    import TxSummary from "@/components/wallet/transfer/TxSummary.vue";
    import {IssueBatchTxInput} from "@/store/types";
    import {bnToBig} from "@/helpers/helper";



    @Component({
        components: {
            FaucetLink,
            TxList,
            RadioButtons,
            QrInput,
            NftList,
            TxSummary
        }
    })
    export default class Transfer extends Vue{
        showAdvanced:boolean = false;
        isAjax:boolean = false;
        addressIn:string = '';
        memo: string = "";
        orders:ITransaction[] = [];
        nftOrders: UTXO[] = [];
        formErrors:string[] = [];
        err = '';

        formAddress:string = '';
        formOrders:ITransaction[] = [];
        formNftOrders: UTXO[] = [];
        formMemo = "";

        isConfirm = false;
        isSuccess = false;
        txId = "";

        confirm(){
            let isValid = this.formCheck();
            if(!isValid) return;


            this.formOrders = [...this.orders];
            this.formNftOrders = [...this.nftOrders];
            this.formAddress = this.addressIn;
            this.formMemo = this.memo;

            this.isConfirm = true;
        }

        cancelConfirm(){
            this.err = '';
            this.formMemo = "";
            this.formOrders = [];
            this.formNftOrders = [];
            this.formAddress = "";
            this.isConfirm = false;
        }

        updateTxList(data:ITransaction[]){
            this.orders = data;
        }

        updateNftList(val: UTXO[]){
            this.nftOrders = val;
        }

        formCheck(){
            this.formErrors = [];
            let err = [];

            let addr = this.addressIn;

            let chain = addr.split('-');

            if(chain[0] !== 'X'){
                err.push('Invalid address. You can only send to other X addresses.')
            }

            if(!isValidAddress(addr)){
                err.push('Invalid address.')
            }

            let memo = this.memo;
            if(this.memo){
                let buff = Buffer.from(memo);
                let size = buff.length;
                if(size>256){
                    err.push('You can have a maximum of 256 characters in your memo.')
                }
            }


            // Make sure to address matches the bech32 network hrp
            let hrp = ava.getHRP();
            if(!addr.includes(hrp)){
                err.push('Not a valid address for this network.')
            }

            this.formErrors = err;
            if(err.length===0){
                // this.send();
                return true;
            }else{
                return false;
            }
        }

        startAgain(){
            this.txId = "";
            this.isSuccess = false;
            this.cancelConfirm();
        }

        clearForm(){
            this.addressIn = "";
            this.memo = "";
            // Clear transactions list
            // @ts-ignore
            this.$refs.txList.clear();

            // Clear NFT list
            if(this.hasNFT){
                // @ts-ignore
                this.$refs.nftList.clear();
            }
        }

        onsuccess(){
            this.isAjax = false;
            this.isSuccess = true;
            this.clearForm();

            this.$store.dispatch('Notifications/add', {
                title: 'Transaction Sent',
                message: 'You have successfully sent your transaction.',
                type:'success',
            });


            // Update the user's balance
            setTimeout(()=>{
                this.$store.dispatch('Assets/updateUTXOs');
            }, 3000);
        }

        onerror(err: any){
            this.err = err;
            this.isAjax = false;
            this.$store.dispatch('Notifications/add', {
                title: 'Error Sending Transaction',
                message: 'Failed to send transaction.',
                type:'error',
            });
        }


        submit(){
            this.isAjax = true;
            this.err = '';

            let sumArray: (ITransaction|UTXO)[] = [...this.formOrders, ...this.formNftOrders];

            let txList: IssueBatchTxInput = {
                toAddress: this.formAddress,
                memo: Buffer.from(this.formMemo),
                orders: sumArray
            };


            this.$store.dispatch('issueBatchTx', txList).then(res => {

                console.log(res);
                this.onsuccess()
                this.txId = res;
            }).catch(err => {
                this.onerror(err);
            });
        }

        get networkStatus():string{
            let stat = this.$store.state.Network.status;
            return stat;
        }

        get hasNFT(): boolean{
            return this.$store.getters.walletNftUTXOs.length > 0;
        }

        get faucetLink(){
            let link = process.env.VUE_APP_FAUCET_LINK;
            if(link) return link;
            return null;
        }
        get canSend(){
            if(!this.addressIn) return false;

            if((this.orders.length > 0 && this.totalTxSize.eq(new BN(0))) && this.nftOrders.length===0 ){
                return false;
            }

            if(this.orders.length === 0 && this.nftOrders.length===0) return false;

            return true;
        }
        get totalTxSize(){
            let res = new BN(0);
            for(var i=0; i<this.orders.length; i++){
                let order = this.orders[i];
                if(order.amount){
                    res = res.add(this.orders[i].amount);
                }
            }
            return res;
        }

        get txFee(): Big{
            let fee = avm.getTxFee();
            return bnToBig(fee,9)
        }

        get addresses(){
            return this.$store.state.addresses;
        }
    }
</script>


<style lang="scss">

    .advanced_panel{
        .v-expansion-panel-header{
            padding: 0;
            font-size: 12px;
            font-weight: normal;
            color: #2c3e50;
            min-height: auto !important;
            margin-bottom: 10px;
        }
        .v-expansion-panel-content__wrap{
            padding: 0 !important;
        }

        .v-icon{
            font-size: 12px;
        }
    }
</style>
<style scoped lang="scss">
    @use '../../main';

    $padLeft: 24px;
    $padTop: 8px;

    .disconnected{
        padding: 30px;
        text-align: center;
        background-color: var(--bg-light);
    }

    .explain{
        font-size: 12px;
        color: var(--primary-color-light);
    }
    h1{
        font-weight: normal;
    }
    h4{
        display: block;
        text-align: left;
        font-size: 12px;
        font-weight: bold;
        margin: 12px 0;
    }


    .send_to{
        display: flex;
        margin-bottom: 10px;
    }

    .addressIn >>> input{
        color: var(--bg) !important;
        padding: 5px 6px !important;
        text-align: center;
        letter-spacing: 2px;
        font-size: 12px;
    }

    .qrIn{
        border-radius: 2px !important;
        height: 40px;
        font-size: 12px;
    }

    .addressIn >>> input::-webkit-input-placeholder{
        color: var(--primary-color-light) !important;
    }

    .addressIn .v-input__slot:before{
        display: none;
    }

    .readerBut{
        margin-top: 4px;
        display: flex;
        background-color: #404040;
        /*cursor: pointer;*/
    }
    .readerBut button{
        opacity: 0.6;
        outline: none;
        padding: 6px 12px;
        margin: 0px auto;
    }
    .readerBut:hover button{
        opacity: 1;
    }

    .memo{
        font-size: 14px;
        background-color: var(--bg-light);
        resize: none;
        width: 100%;
        height: 80px;
        border-radius: 2px;
        padding: 4px 12px;
    }

    .radio_buttons{
        margin-top: 15px;
    }



    .tx_info{
        text-align: left;
        font-size: 14px;
    }

    .new_order_Form{
        display: grid;
        grid-template-columns: 1fr 1fr 300px;
        column-gap: 45px;
        padding-top: 15px;
    }

    .new_order_Form > div{
        /*padding: 10px 0;*/
        margin-bottom: 15px;
    }
    .lists{
        /*padding-right: 45px;*/
        border-right: 1px solid var(--bg-light);
        grid-column: 1/3;

        /*> div{*/
        /*    margin: 14px 0;*/
        /*}*/
    }

    .tx_list{
        margin-bottom: 14px;
    }

    .fees p{
        text-align: left;
        font-size: 13px;
        color: var(--primary-color-light);
    }

    .fees span{
        float: right;
    }

    .to_address {
        margin-bottom: 14px;
        border-bottom: 1px solid var(--bg-light);
        padding-bottom: 14px;
    }

    label{
        color: var(--primary-color-light);
        font-size: 12px;
        font-weight: bold;
        margin: 2px 0 !important;
    }

    .faucet{
        margin-top: 20px;
    }

    .advanced{
        padding: 20px 0px !important;
        margin-bottom: 20px;
    }


    .advanced .advancedBody{
        transition-duration: 0.2s;
    }


    .err_list{
        font-size: 12px;
        color: var(--error);
        margin: 6px 0;
    }

    .checkout{
        margin-top: 14px;
    }

    .confirm_val{
        background-color: var(--bg-light);
        word-break: break-all;
        padding: 8px 16px;
    }

    @media only screen and (max-width: 600px) {
        .order_form{
            display: block;
        }
        .asset_select button{
            flex-grow: 1;
            word-break: break-word;
        }
    }

    @media only screen and (max-width: main.$mobile_width) {
        .transfer_card {
            display: block;
            grid-template-columns: none;
        }

        .but_primary{
            width: 100%;
        }

        .new_order_Form{
            display: block;
            grid-template-columns: none;
        }

        .tx_list{
            padding: 0;
            border: none;
        }
    }
</style>
