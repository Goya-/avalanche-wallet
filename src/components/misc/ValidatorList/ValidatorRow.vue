<template>
    <tr class="validator_row" v-if="isVisible">
        <td class="id">{{validator.nodeID}}</td>
        <td class="amount">{{amtText}}</td>
        <td class="amount">{{remainingAmtText}}</td>
        <td style="text-align: center;">{{numDelegators}}</td>
        <td>{{remainingTimeText}}</td>
        <td>{{uptimeText}}</td>
        <td>{{feeText}}%</td>
        <td>
            <button class="button_secondary" @click="select">Select</button>
        </td>
    </tr>
</template>
<script lang="ts">
import "reflect-metadata";
import { Vue, Component, Prop } from "vue-property-decorator";
import {ava, pChain} from "@/AVA";
import {DelegatorRaw, ValidatorRaw} from "@/components/misc/ValidatorList/types";
import moment from "moment";
import Big from "big.js";
import {BN} from "avalanche";
import {bnToBig} from "@/helpers/helper";

@Component({
})
export default class ValidatorsList extends Vue{
    @Prop() validator!: ValidatorRaw;

    get remainingMs():number{
        let end = parseInt(this.validator.endTime);
        let remain =  end*1000 - Date.now();
        return remain;
    }

    get remainingTimeText(){
        let ms = this.remainingMs;
        let sec = ms/1000;

        let duration = moment.duration(ms, 'milliseconds');
        return duration.humanize(true)
    }

    get stakeAmt(): BN {
        return new BN(this.validator.stakeAmount);
    }
    get amtText(){
        let amt = this.stakeAmt;
        let big = bnToBig(amt,9);
        return big.toLocaleString(0);
    }

    get uptimeText(): string{
        let uptime = parseFloat(this.validator.uptime) * 100;

        // if(!uptime) return '?';

        return uptime.toFixed(2) + ' %';
    }

    get feeText(){
        return parseFloat(this.validator.delegationFee);
    }

    get delegators(): DelegatorRaw[]{
        let map = this.$store.getters["Platform/nodeDelegatorMap"];
        return map[this.validator.nodeID];
    }

    get numDelegators(){
        if(!this.delegators) return 0;
        return this.delegators.length;
    }

    get totalDelegated(): BN{
        return this.$store.getters["Platform/validatorTotalDelegated"](this.validator.nodeID);
    }

    get maxStake(): BN{
        return this.$store.getters["Platform/validatorMaxStake"](this.validator);
    }


    get remainingStake(): BN{
        return this.maxStake.sub(this.totalDelegated.add(this.stakeAmt));
    }


    get remainingAmtText(): string{
        let big = bnToBig(this.remainingStake, 9);
        return big.toLocaleString(0)
    }

    get isVisible(){

        // If remaining amount is less than the minimum delegation amount
        let minDelAmt = this.$store.state.Platform.minStakeDelegation;
        if(this.remainingStake.lt(minDelAmt)) return false;

        return true;
    }

    select(){
        this.$emit('select', this.validator);
    }
}
</script>
<style scoped lang="scss">
//.validator_row{
//    padding: 4px 0;
//    display: grid;
//    grid-gap: 14px;
//    grid-template-columns: 1fr max-content max-content max-content max-content;
//}


.amount{
    text-align: right;
    font-family: monospace;
}

button{
    padding: 3px 12px;
    font-size: 13px;
    border-radius: 3px;
}

.id{
    word-break: break-all;
}
td{
    padding: 4px 14px;
    background-color: var(--bg-light);
    border: 1px solid var(--bg);
    font-size: 14px;
}
</style>
