<template>
    <div class="create_wallet">
        <b-container>
            <b-row>
                <b-col>
                    <transition name="fade" mode="out-in">
                        <!-- PHASE 1 -->
                        <div v-if="!keyPhrase" class="stage_1">
                            <div class="img_container">
                                <img v-if="$root.theme === 'day'" src="@/assets/diamond-secondary.png" alt />
                                <img v-else src="@/assets/diamond-secondary-night.svg" alt />
                            </div>
                            <h1>Generate a new key phrase to setup your Avalanche Wallet.</h1>
                            <router-link to="/access" class="link">Already have a wallet?</router-link>
                            <div class="options">
                                <button class="ava_button but_generate button_secondary" @click="createKey">Generate Key Phrase</button>
<!--                                <TorusGoogle class="torus_but"></TorusGoogle>-->
                            </div>
                            <router-link to="/" class="link">Cancel</router-link>
                        </div>
                        <!-- PHASE 2 -->
                        <div v-else class="stage_2">
                            <div class="cols">
                                <!-- LEFT -->
                                <div class="mneumonic_disp_col">
                                    <div class="mnemonic_disp">
                                        <mnemonic-display :phrase="keyPhrase" :bgColor="verificatiionColor" class="mnemonic_display"></mnemonic-display>
                                        <p class="phrase_raw" v-bind:class="{ verified: isVerified }">{{keyPhrase}}</p>
                                        <div class="mneumonic_button_container" v-if="!isVerified">
                                            <button @click="createKey" class="ava_button but_randomize button_primary ">
                                                <fa icon="sync"></fa>
                                                <span>Regenerate</span>
                                            </button>
                                        </div>
                                    </div>
                                </div>
                                <!-- RIGHT -->
                                <div class="phrase_disp_col">
                                    <template v-if="!isVerified">
                                        <img v-if="$root.theme === 'day'" src="@/assets/keyphrase.png" alt />
                                        <img v-else src="@/assets/keyphrase_night.svg" alt />
                                    </template>
                                    <template v-else>
                                        <img src="@/assets/success.svg" alt />
                                    </template>
                                    <header v-if="!isVerified">
                                        <h1>This is your key phrase.</h1>
                                        <p>Use these 24 words in sequential order to recover your Avalanche Wallet</p>
                                    </header>
                                    <header v-else>
                                        <h1>Congratulations!</h1>
                                        <p>It's time to access your Avalanche Wallet.</p>
                                    </header>
                                    <p class="warn" v-if="!isVerified">
                                        <span class="label">Attention!</span>
                                        <span class="description">Store this key phrase in a secure location. Anyone with this key phrase can access your Avalanche Wallet. There is no way to recover lost key phrases.</span>
                                    </p>
                                    <!-- STEP 2a - VERIFY -->
                                    <div class="verify_cont" v-if="!isVerified">
                                        <MnemonicCopied
                                                v-model="isSecured"
                                                explain="I wrote down my key phrase in a secure location."
                                        ></MnemonicCopied>
                                        <VerifyMnemonic :mnemonic="keyPhrase" ref="verify" @complete="complete"></VerifyMnemonic>
                                        <button class="but_primary ava_button button_secondary" @click="verifyMnemonic" :disabled="!canVerify">Verify</button>
                                    </div>
                                    <!-- STEP 2b - ACCESS -->
                                    <div class="access_cont" v-if="isVerified">
<!--                                        <remember-key-->

<!--                                                v-model="rememberPassword"-->
<!--                                                @is-valid="isRememberValid"-->
<!--                                                class="remember_wallet"-->
<!--                                                complete="onremember"-->
<!--                                                explain="Remember my wallet."-->
<!--                                        ></remember-key>-->

                                        <div class="submit">
                                            <transition name="fade" mode="out-in">
                                                <Spinner v-if="isLoad" class="spinner"></Spinner>
                                                <div v-else>
                                                    <button
                                                            class="button_primary ava_button access generate"
                                                            @click="access"
                                                            :disabled="!canSubmit"
                                                    >Access Wallet</button>
                                                    <router-link to="/" class="link">Cancel</router-link>
                                                    <ToS style="margin: 30px 0 !important;"></ToS>
                                                </div>
                                            </transition>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </transition>
                </b-col>
            </b-row>
        </b-container>
        <div>
        </div>
    </div>
</template>
<script lang="ts">
    import 'reflect-metadata';
    import { Vue, Component, Prop } from 'vue-property-decorator';
    import TextDisplayCopy from "@/components/misc/TextDisplayCopy.vue";
    import Spinner from '@/components/misc/Spinner.vue';
    // import RememberKey from "@/components/misc/RememberKey.vue";
    // import TorusGoogle from "@/components/Torus/TorusGoogle.vue";
    import MnemonicDisplay from "@/components/misc/MnemonicDisplay.vue";
    import CopyText from "@/components/misc/CopyText.vue";
    import * as bip39 from 'bip39';

    import VerifyMnemonic from "@/components/CreateWalletWorkflow/VerifyMnemonic.vue";
    import MnemonicCopied from "@/components/CreateWalletWorkflow/MnemonicCopied.vue";
    import ToS from "@/components/misc/ToS.vue";


    @Component({
        components: {
            ToS,
            CopyText,
            // RememberKey,
            TextDisplayCopy,
            MnemonicDisplay,
            Spinner,
            // TorusGoogle,
            VerifyMnemonic,
            MnemonicCopied
        }
    })
    export default class CreateWallet extends Vue{
        // TODO: We do not need to create keyPair, only mnemonic is sufficient
        isLoad: boolean = false;
        // rememberPassword:string|null = null;
        // rememberValid:boolean = true;         // Will be true if the values in remember wallet checkbox are valid
        // Mnemonic
        // newPrivateKey: string|null =null;
        keyPhrase: string = "";
        // keyPair: KeyPair|null = null;
        // Verify Mnemonic
        isSecured: boolean = false;
        isVerified: boolean = false;


        get canVerify(): boolean{
            return this.isSecured ? true : false;
        }

        get verificatiionColor() {
            return this.isVerified ? '#a9efbf' : '#F5F6FA';
        }

        createKey():void{
            this.isSecured = false;
            let mnemonic = bip39.generateMnemonic(256);
            this.keyPhrase = mnemonic;
        }

        // Will be true if the values in remember wallet checkbox are valid
        // isRememberValid(val: boolean){
        //     this.rememberValid = val;
        // }

        get canSubmit():boolean{
            // if(!this.rememberValid) return false;
            return true;
        }
        verifyMnemonic(){
            // @ts-ignore
            this.$refs.verify.open();
        }

        complete(){
            this.isVerified = true;
        }

        async access(): Promise<void> {
            if (!this.keyPhrase) return;

            this.isLoad = true;


            let parent = this;

            setTimeout(async ()=>{
                await parent.$store.dispatch('accessWallet', this.keyPhrase);

                // if(this.rememberPassword && this.rememberValid){
                //     // console.log("Will remember..");
                //     parent.$store.dispatch('rememberWallets', this.rememberPassword);
                // }
            }, 500);
        }
    }
</script>
<style scoped lang="scss">
@use '../../main';

.create_wallet {
    display: flex;
    justify-content: center;
    align-items: center;
}

/* ==========================================
   stage_1
   ========================================== */

.stage_1 {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: space-between;
    background-color: var(--bg-light);
    padding: main.$container-padding;
    text-align: center;
    /*min-width: 1000px;*/

    img {
        margin-top: main.$vertical-padding;
        width: 89px;
        height: 89px;
        max-height: none;
    }

    h1 {
        margin-top: main.$vertical-padding;
        text-align: left;
        font-size: main.$m-size;
        font-weight: 400;
    }


}

.options{
    display: flex;
    flex-direction: row;
    align-items: center;
    margin: 15px;
    padding-top: 15px;
    border-top: 1px solid var(--primary-color-light);

    >*{
        margin: 4px;
        font-size: 0.8rem;
    }

    p{
        color: #999;
        margin: 6px !important;
    }
}

.torus_but{
    background-color: #db3236;
    color: #FFF;
}


.but_generate {
    display: block;
    height: max-content;
    background-color: main.$secondary-color;
}


.key_disp {
    margin: 30px auto;
    font-size: 12px;
}

a {
    color: main.$primary-color-light !important;
    text-decoration: underline !important;
    margin-top: 10px;
}

/* ==========================================
   mneumonic
   ========================================== */

.stage_2 {
    margin: 0 auto;
    text-align: left;
    align-items: flex-start;
}

.cols {
    display: grid;
    grid-template-columns: 1fr 1fr;
    column-gap: 60px;
}

.mneumonic_disp_col {
    .mnemonic_disp {
        max-width: 560px;
        justify-self: center;
        display: flex;
        flex-direction: column;
    }

    .phrase_raw {
        color: var(--primary-color);
        background-color: var(--bg-light);
        padding: 14px 24px;
        text-align: justify;
        border-radius: 4px;
        margin: 30px 0px !important;
    }

    .mnemonic_display{
        background-color: var(--bg-light);
        padding: 14px;
    }

    .verified {
        background-color: main.$green-light;
        color: #222;
    }

    .mneumonic_button_container {

        .but_randomize {
            span {
                margin-left: 12px;
            }
        }
    }
}

.phrase_disp_col {
    padding: 0 30px;
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    justify-content: center;

    > * {
        width: 100%;
    }

    img {
        width: main.$img-size;
        height: main.$img-size;
        max-height: none;
    }

    header {
        h1 {
            margin-top: 10px;
            font-size: main.$xl-size;
            line-height: 1.25em;
            font-weight: 400;
        }

        p {
            color: main.$primary-color-light;
        }
    }

    .warn {
        margin-top: main.$vertical-padding !important;

        span {
            display: block;
            font-size: main.$s-size;
            font-weight: 700;
            text-transform: uppercase;

            &.label {
                color: main.$secondary-color;
                text-transform: uppercase;
            }

            &.description {
                color: main.$primary-color-light !important;
            }
        }
    }

    .access_cont {
        text-align: left;
        flex-direction: column;

        .submit {
            display: flex;
            flex-direction: row;
            margin-top: 14px;
            text-align: left;
            //flex-direction: column;
            //align-items: flex-start;
            //justify-content: space-between;

            .access {

            }

            .link {
                margin-left: 40px;
            }
        }
    }
}

.spinner{
    width: 26px !important;
    margin: 0px auto;
}



.remember_wallet{
    margin: 20px 0;
}




@include main.medium-device {
    .stage_1 {
        min-width: unset;
    }
}

@include main.mobile-device {
    .stage_1 {
        min-width: unset;
    }

    .stage_2 {
        min-width: unset;
    }

    .access {
        margin: 30px auto;
        width: 100%;
    }

    .cols {
        display: block;
    }

    .options{
        margin: 30px 0px;
        flex-direction: column;

        > button{
            width: 100%;
        }
    }

    .mneumonic_disp_col {
        .mnemonic_disp {
            margin: 0 auto;
        }

        .mneumonic_button_container {
            display: flex;
            flex-direction: column;
            align-items: flex-start;
            justify-content: center;

            .copy_phrase {
                margin-right: 0;
                width: 100%;
                display: flex;
                flex-direction: row;
                justify-content: center;
            }

            .but_randomize {
                margin-top: 10px;

                span {
                    margin-left: 12px;
                }
            }
        }
    }

    .phrase_disp_col {
        padding: 30px 0;
        align-items: center;

        img {
            width: main.$img-size-mobile;
            height: main.$img-size-mobile;
        }

        header {
            h1 {
                font-size: main.$xl-size-mobile;
            }
        }

        .warn {
            margin-top: main.$vertical-padding-mobile !important;
        }

        .access_cont {
            .submit {
                flex-direction: column;
                justify-content: center;

                > div {
                    display: flex;
                    flex-direction: column;
                    justify-content: center;
                    align-items: center;
                }

                .link {
                    margin: auto;
                }
            }
        }
    }
}
</style>
<style lang="scss">
    .create_wallet{
        .remember_wallet{
            .v-expansion-panel-header, .v-expansion-panel-content__wrap{
                padding: 6px 0;
            }
        }
    }
</style>
