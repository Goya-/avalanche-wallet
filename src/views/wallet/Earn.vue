<template>
    <div>
        <div class="header">
            <h1>Earn </h1>
            <h1 class="subtitle" v-if="pageNow">/ {{subtitle}} <span @click="cancel"><fa icon="times"></fa></span></h1>
        </div>
        <transition name="fade" mode="out-in">
            <div v-if="!pageNow">
                <p>You can earn more AVAX by staking your existing tokens.</p>
                <div class="options">
                    <div>
                        <h4 class="title">Validate</h4>
                        <p style="flex-grow: 1">You have an Avalanche node that you want to stake with.</p>
                        <p v-if="!canValidate" class="no_balance">You must have at least <b>{{minStakeAmt.toLocaleString()}} AVAX</b> on the P chain to become a validator.</p>
                        <v-btn class="button_secondary" data-cy="validate" @click="addValidator" depressed small :disabled="!canValidate">Add Validator</v-btn>
<!--                        <v-btn class="button_secondary" depressed small disabled>Coming Soon</v-btn>-->
                    </div>
                    <div>
                        <h4 class="title">Delegate</h4>
                        <p style="flex-grow: 1">You do not own an Avalanche node, but you want to stake using another node.</p>
                        <p v-if="!canDelegate" class="no_balance">You must have at least <b>{{minDelegationAmt.toLocaleString()}} AVAX</b> on the P chain to become a delegator.</p>
                        <v-btn class="button_secondary" data-cy="delegate" @click="addDelegator" depressed small :disabled="!canDelegate">Add Delegator</v-btn>
<!--                        <v-btn class="button_secondary" depressed small disabled>Coming Soon</v-btn>-->
                    </div>
                    <div>
                        <h4 class="title">Cross Chain Transfer</h4>
                        <p style="flex-grow: 1">Staking requires AVAX on the P chain. Transfer tokens between X and P.</p>
                        <v-btn class="button_secondary" data-cy="swap" @click="transfer" depressed small>Transfer</v-btn>
                    </div>
                    <div>
                        <h4 class="title">Estimated Rewards</h4>
                        <p style="flex-grow: 1">View staking rewards you will receive.</p>
                        <v-btn class="button_secondary" data-cy="rewards" @click="viewRewards" depressed small>View Rewards</v-btn>
                    </div>
                </div>
<!--                <v-btn @click="viewRewards" depressed small>View Estimated Rewards</v-btn>-->
            </div>
            <div v-else>
                <component  :is="pageNow" class="comp" @cancel="cancel"></component>
            </div>
        </transition>
    </div>
</template>
<script lang="ts">
import "reflect-metadata";
import { Vue, Component, Prop } from "vue-property-decorator";

import AddValidator from "@/components/wallet/earn/Validate/AddValidator.vue";
import AddDelegator from "@/components/wallet/earn/Delegate/AddDelegator.vue";
import ChainTransfer from "@/components/wallet/earn/ChainTransfer.vue";
import {BN} from "avalanche/dist";
import UserRewards from "@/components/wallet/earn/UserRewards.vue";
import {bnToBig} from "@/helpers/helper";
import Big from 'big.js';

@Component({
    name: "earn",
    components: {
        UserRewards,
        AddValidator,
        AddDelegator,
        ChainTransfer
    }
})
export default class Earn extends Vue{
    pageNow: any = null;
    subtitle: string = '';
    intervalID: any = null;

    addValidator(){
        this.pageNow = AddValidator;
        this.subtitle = "Validate"
    }
    addDelegator(){
        this.pageNow = AddDelegator;
        this.subtitle = "Delegate"
    }
    transfer(){
        this.pageNow = ChainTransfer;
        this.subtitle = "Cross Chain Transfer"
    }

    viewRewards(){
        this.pageNow = UserRewards
        this.subtitle = "Estimated Rewards";
    }
    cancel(){
        this.pageNow = null;
        this.subtitle = '';
    }

    updateValidators(){
        this.$store.dispatch('Platform/update')
    }

    created(){
        this.updateValidators();
        this.intervalID = setInterval(()=>{
            this.updateValidators();
        },15000);
    }

    destroyed(){
        clearInterval(this.intervalID);
    }

    get platformUnlocked(): BN{
        return this.$store.getters.walletPlatformBalance;
    }

    get platformLockedStakeable(): BN{
        return this.$store.getters.walletPlatformBalanceLockedStakeable;
    }

    get totBal(): BN{
        return this.platformUnlocked.add(this.platformLockedStakeable);
    }

    get pNoBalance(){
        return this.platformUnlocked.add(this.platformLockedStakeable).isZero();
    }

    get canDelegate(): boolean{
        let bn = this.$store.state.Platform.minStakeDelegation;
        if(this.totBal.lt(bn)){
            return false;
        }
        return true;
    }

    get canValidate(): boolean{
        let bn = this.$store.state.Platform.minStake;
        if(this.totBal.lt(bn)){
            return false;
        }
        return true;
    }

    get minStakeAmt(): Big{
        let bn = this.$store.state.Platform.minStake;
        return bnToBig(bn,9)
    }

    get minDelegationAmt(): Big{
        let bn = this.$store.state.Platform.minStakeDelegation;
        return bnToBig(bn,9)
    }
}
</script>
<style scoped lang="scss">
@use '../../main';
.header{
    display: flex;
    /*justify-content: space-between;*/
    /*align-items: center;*/
    align-items: center;

    .subtitle{
        margin-left: 0.5em;
        /*font-size: 20px;*/
        color: var(--primary-color-light);
        font-weight: lighter;
    }

    span{
        margin-left: 1em;

        &:hover{
            color: var(--primary-color);
            cursor: pointer;
        }
    }
}
    .options{
        margin: 30px 0;
        display: grid;
        grid-template-columns: 1fr 1fr 1fr 1fr;
        grid-gap: 14px;
        //display: flex;
        //justify-content: space-evenly;
        //padding: 60px;

        >div{
            width: 100%;
            justify-self: center;
            display: flex;
            flex-direction: column;
            justify-content: flex-start;
            align-items: flex-start;
            //max-width: 260px;
            padding: 30px;
            border-radius: 4px;
            background-color: var(--bg-light);
        }

        h4{
            font-size: 32px !important;
            font-weight: lighter;
            color: var(--primary-color-light)
        }

        p{
            /*color: var(--primary-color-light);*/
            margin: 14px 0 !important;
        }

        .no_balance{
            color: var(--secondary-color);
        }

        .v-btn{
            margin-top: 14px;
        }
    }

    span{
        color: var(--primary-color-light);
        opacity: 0.5;
        float: right;
        font-weight: lighter;
    }

    .cancel{
        font-size: 13px;
        color: var(--secondary-color);
        justify-self: flex-end;
    }

    .comp{
        margin-top: 14px;
    }


    @include main.medium-device{
        .options{
            grid-template-columns: 1fr 1fr;
        }
    }

    @include main.mobile-device{
        .options{
            grid-template-columns: none;
            grid-row-gap: 15px;
        }
    }
</style>
