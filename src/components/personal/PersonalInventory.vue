<template>
  <div>
    <table class="table table-hover w-100">
      <thead>
      <tr>
        <th/>
        <th class="text-start">Item<SortIcon field="name"/></th>
        <th class="text-end">Balance<SortIcon field="balance"/></th>
        <th class="text-end">Jewel Price<SortIcon field="price"/></th>
        <th class="text-end">Jewels<SortIcon field="jewels"/></th>
        <th class="text-end">Jewel USD</th>
        <th class="text-end">Gold Price<SortIcon field="priceGold"/></th>
        <th class="text-end">Gold<SortIcon field="gold"/></th>
        <th class="text-end">Gold USD</th>
      </tr>
      </thead>
      <tbody>
      <tr v-for="item in sortedItems" :key="item">
        <td><img height="25" width="25" :src="item.logoURI" :alt="item.name" /></td>
        <td class="text-start">{{ item.name }}</td>
        <td class="text-end">{{ item.balance }}</td>
        <td class="text-end"><span v-if="item.jewelPrice">{{ formatNumber(item.jewelPrice) }}</span></td>
        <td class="text-end"><span v-if="item.jewelPrice">{{ formatNumber(item.totalJewel) }}</span></td>
        <td class="text-end" :class="betterJewelTrade(item)">
          <span v-if="item.jewelPrice">{{ formatNumber(item.totalJewelUsd, '$') }}</span>
        </td>
        <td class="text-end"><span v-if="item.goldPrice">{{ formatNumber(item.goldPrice) }}</span></td>
        <td class="text-end"><span v-if="item.goldPrice">{{ formatNumber(item.totalGold) }}</span></td>
        <td :class="betterGoldTrade(item)" class="text-end ">
          <span v-if="item.goldPrice">{{formatNumber(item.totalGoldUsd, '$') }}</span>
        </td>
      </tr>
      </tbody>
      <tfoot>
      <tr>
        <th class="text-end" colspan="5">{{ formatNumber(totalJewels) }}</th>
        <th class="text-end">{{ formatNumber(totalJewelUsd, '$') }}</th>
        <th class="text-end" colspan="2">{{ formatNumber(totalGold) }}</th>
        <th class="text-end">{{ formatNumber(totalGoldUsd, '$') }}</th>
      </tr>
      </tfoot>

    </table>
  </div>
</template>

<script>
import formatNumber from "@/utils/FormatNumber";
import SortIcon from "@/components/generic/SortIcon";
import { getItem, GetTokenList, getAllItemAddresses } from "@/utils/Items";
import _ from 'lodash'

import {formatUnits, contractJson, RPCs} from "@/utils/ethers"
import { Contract } from 'ethers'
import axios from "axios";

const defaultSort = {
  name: 0,
  balance: 0,
  price: 0,
  jewels: 0,
  priceGold: 0,
  gold: 0,
}

const excludedItems = [
  "JEWEL", "AVAX", "CRYSTAL", "CRYSTAL-AVAX", "JEWEL-AVAX", "JEWEL-CRYSTAL", "JEWEL-xJEWEL",
  "WJEWEL", "xJEWEL", "xCRYSTAL"
]


const GOLD_ADDR = "0x3a4EDcf3312f44EF027acfd8c21382a5259936e7"

export default {
  name: "PersonalInventory",
  components: {SortIcon},
  props: ["userAddress"],
  inject: ["prices", "setInventoryTotal"],
  data() {
    return {
      items: [],
      itemSort: {...defaultSort}
    }
  },
  computed: {
    sortedItems() {
      const sortOrder = this.itemSort
      const items = [...this.items]

      if (sortOrder.balance > 0)
        items.sort((a, b) => a.balance - b.balance)
      else if (sortOrder.balance < 0)
        items.sort((a, b) => b.balance - a.balance)

      else if (sortOrder.name > 0)
        items.sort((a, b) => a.name.localeCompare(b.name))
      else if (sortOrder.name < 0)
        items.sort((a, b) => b.name.localeCompare(a.name))

      else if (sortOrder.price > 0)
        items.sort((a, b) => a.price - b.price)
      else if (sortOrder.price < 0)
        items.sort((a, b) => b.price - a.price)

      else if (sortOrder.jewels > 0)
        items.sort((a, b) => a.jewels - b.jewels)
      else if (sortOrder.jewels < 0)
        items.sort((a, b) => b.jewels - a.jewels)

      else if (sortOrder.priceGold > 0)
        items.sort((a, b) => a.goldPrice - b.goldPrice)
      else if (sortOrder.priceGold < 0)
        items.sort((a, b) => b.goldPrice - a.goldPrice)

      else if (sortOrder.gold > 0)
        items.sort((a, b) => a.gold - b.gold)
      else if (sortOrder.gold < 0)
        items.sort((a, b) => b.gold - a.gold)

      else
        items.sort((a, b) => a.name.localeCompare(b.name))


      return items
    },
    totalJewels() {
      let jewels = _.sumBy(this.items, "totalJewel")

      this.setInventoryTotal(jewels, 'sd')
      return jewels
    },
    totalGold() {
      return _.sumBy(this.items, "totalGold")
    },
    jewelPrice() {
      return this.prices()["JEWEL"]
    },
    totalJewelUsd() {
      return _.sumBy(this.items, "totalJewelUsd")
    },
    totalGoldUsd() {
      return _.sumBy(this.items, "totalGoldUsd")
    },
    goldItem() {
      return this.items.find(item => item.address === GOLD_ADDR)
    }
  },
  methods: {
    formatNumber(num, prefix, suffix, decimals) {
      return formatNumber(num, prefix, suffix, decimals)
    },
    async loadPrice(address) {
      const dsPrefix = "https://api.dexscreener.io/latest/dex/tokens/"
      let r = await axios.get(dsPrefix + address)
      if (r.status === 200) {
        let pairs = r.data.pairs.filter(pair =>
            pair.dexId === "defikingdoms"
            && Number(pair.priceUsd) > 0
        )
        if (pairs.length > 0)
          return Number(pairs[0].priceUsd)
      } else {
        console.log(`Got status ${r.status} : ${r.statusText} while loading prices`)
      }
      return 0
    },
    async loadInventory() {
      const allItems = []
      // load gold first
      await this.loadItem(GOLD_ADDR)
      for (let address of getAllItemAddresses()) {
        if (address !== GOLD_ADDR) {
          allItems.push(this.loadItem(address))
        }
      }
      await Promise.all(allItems)
    },
    async loadItem(address) {
      const contract = new Contract(address, contractJson.erc20.abi, RPCs.sd)
      const decimals = await contract.decimals()

      const balance = Number(formatUnits(await contract.balanceOf(this.userAddress), decimals))
      if (balance === 0)
        return
      let usdPrice = await this.loadPrice(address)
      let preItem = getItem(address)
      if (excludedItems.includes(preItem.symbol)) {
        return
      }
      const item = {
        ...preItem,
        usdPrice,
        balance,
        jewelPrice: usdPrice/this.jewelPrice,
        totalGold: balance * (address===GOLD_ADDR?1:preItem.goldPrice),
        totalJewel: usdPrice / this.jewelPrice * balance,
        totalGoldUsd: balance * (address===GOLD_ADDR?1:preItem.goldPrice) * (address===GOLD_ADDR?usdPrice:this.goldItem.usdPrice),
        totalJewelUsd: usdPrice * balance
      }
      this.items.push(item)
    },
    toggleSort(field) {
      let currentDir = this.itemSort[field]

      if (currentDir === 0)
        currentDir = -1

      this.itemSort = {...defaultSort}

      this.itemSort[field] = currentDir * -1
    },
    betterJewelTrade(item) {
      return item.totalJewelUsd > item.totalGoldUsd ? "Uncommon" : ""
    },
    betterGoldTrade(item) {
      return (item.totalJewelUsd < item.totalGoldUsd) ? "Uncommon" : ""
    }
  },
  provide() {
    return {
      sortToggle: (field) => this.toggleSort(field)
    }
  },
  mounted() {
    GetTokenList()
    this.loadInventory()
    this.toggleSort("name")
  }
}
</script>

<style scoped>
.Uncommon { color: #14C25A }
</style>
