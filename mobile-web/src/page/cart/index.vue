<template>
  <div class="card">
    <headerNav title="购物车"/>
       <van-cell  value="编辑商品" class="head">
        <template slot="title">
          <van-checkbox v-model="checkedAll" @change="onSelectAll" >全选</van-checkbox>
        </template>
      </van-cell>

    <van-checkbox-group class="card-goods" v-model="checkedItemIds" @change="onItemSelectedChange">

      <div class="promotion-group">
<!--        <div  v-for="(item,index) in goods" :key="index" class="card-goods__item">-->
<!--          <van-checkbox :name="item.id" />-->

<!--          <product-card :product='item' :iscard='false' >-->

<!--          </product-card>-->
<!--        </div>-->

          <div v-for="(itemGroup, i) in itemGroups" class="card-goods__item">
              <van-cell >
                  <template v-if="itemGroup.activity" slot="title">
                      <van-tag type="danger">满减送</van-tag>
                      <span class="van-cell-text" >  {{ formatFullPrivilegeText(itemGroup.activity) }} </span>
                  </template>
              </van-cell>
              <div class="card" v-for="(item, j) in itemGroup.items" :key="j">
                  <van-checkbox :key="item.id" :name="item.id" v-model="item.selected" style="position: relative;top: 40px;" />
                  <product-card :product='convertProduct(item)'/>
                  <van-cell title="优惠信息">
                      {{ formatTimeLimitedDiscountText(item.activity) }}
                  </van-cell>
              </div>
              <div style="height:15px;"></div>
          </div>

      </div>

        <!--      <div class="promotion-group">-->

<!--       <van-cell  is-link class="head">-->
<!--        <template slot="title">-->
<!--          <van-checkbox v-model="checkedAll" >京东自营</van-checkbox>-->
<!--        </template>-->
<!--      </van-cell>-->
<!--        </div>-->
    </van-checkbox-group>

    <div style="height:50px;"></div>
    <van-submit-bar
      :tip="this.formatItemGroupDiscountPriceText()"
      :price="fee.presentTotal"
      :disabled="!checkedItemIds || !checkedItemIds.length"
      :button-text="submitBarText"
      @submit="onSubmit"
    >
    <template slot>
      <van-checkbox v-model="checkedAll" @click="onSelectAll">全选</van-checkbox>
    </template>
    </van-submit-bar>
  </div>
</template>

<script>

import {listCart, updateCartSelected} from "../../api/order";

export default {
  components: {
  },
  data() {
    return {
      itemGroups: [],
      fee: {
        originalTotal: undefined,
        discountTotal: undefined,
        postageTotal: undefined,
        presentTotal: undefined,
      },
      checkedItemIds: undefined, // 通过计算得出
      oldCheckedItemIds: undefined, // 因为 vue 是双向绑定，用于解决 change 的时候，拿不到老值
      checkedAll: undefined, // 通过计算得出
    }
  },
  computed: {
    submitBarText() {
      const count = this.checkedItemIds ? this.checkedItemIds.length : 0;
      return '结算' + (count ? `(${count})` : '');
    },
  },
  methods: {
    formatFullPrivilegeText(activity) {
      if (!activity) {
        return '';
      }
      let text = '';
      let fullPrivilege = activity.fullPrivilege;
      for (let i in fullPrivilege.privileges) {
        let privilege = fullPrivilege.privileges[i];
        if (i > 0) {
          text += ';';
        }
        if (fullPrivilege.cycled) {
          text += '每';
        }
        if (privilege.meetType === 1) {
          text += '满 ' + privilege.meetValue / 100.0 + ' 元,';
        } else if (privilege.meetType === 2) {
          text += '满 ' + privilege.meetValue + ' 件,';
        }
        if (privilege.preferentialType === 1) {
          text += '减 ' + privilege.preferentialValue / 100.0 + ' 元';
        } else if (privilege.preferentialType === 2) {
          text += '打 ' + privilege.preferentialValue / 10.0 + ' 折';
        }
      }
      return text;
    },
    formatTimeLimitedDiscountText(activity) {
      if (!activity) {
        return '';
      }
      let text = '';
      let timeLimitedDiscount = activity.timeLimitedDiscount.items[0];
      if (timeLimitedDiscount.preferentialType === 1) {
        text += '减 ' + timeLimitedDiscount.preferentialValue / 100.0 + ' 元';
      } else if (timeLimitedDiscount.preferentialType === 2) {
        text += '打 ' + timeLimitedDiscount.preferentialValue / 10.0 + ' 折';
      }
      if (activity.timeLimitedDiscount.quota > 0) {
        text += '【限购 ' + activity.timeLimitedDiscount.quota + ' 件】';
      }
      return text;
    },

    formatItemGroupDiscountPriceText() {
      // let price = 0;
      // for (let i in this.itemGroups) {
      //   let itemGroup = this.itemGroups[i];
      //   price += itemGroup.activityDiscountTotal || 0;
      // }
      return this.fee.discountTotal > 0 ? '立减 ' + this.fee.discountTotal / 100.0 + ' 元' : '';
    },

    calCheckedItemIds() {
      // debugger;
      let itemIds = [];
      let checkedAll = true;
      for (let i in this.itemGroups) {
        let items = this.itemGroups[i].items;
        for (let j in items) {
          if (items[j].selected) {
            itemIds.push(items[j].id);
          } else {
            checkedAll = false;
          }
        }
      }
      // 赋值给 checkedItemIds、oldCheckedItemIds、checkedAll
      this.checkedItemIds = itemIds;
      this.oldCheckedItemIds = itemIds;
      this.checkedAll = checkedAll;
    },
    getItemIds() {
      let itemIds = [];
      for (let i in this.itemGroups) {
        let items = this.itemGroups[i].items;
        for (let j in items) {
          itemIds.push(items[j].id);
        }
      }
      return itemIds;
    },
    handleData(data) {
      this.itemGroups = data.itemGroups;
      this.fee = data.fee;
      // 计算 checkedItemIds + checkedAll
      this.calCheckedItemIds();
    },
    onItemSelectedChange(newVal) { // TODO 芋艿，后续研究下。这样的处理方式，很奇怪。
      if (!this.checkedItemIds) {
        return;
      }
      let selected;
      let diffItemIds;
      if (newVal.length > this.oldCheckedItemIds.length) { // 新增
        selected = true;
        let that = this;
        diffItemIds = [...newVal].filter(function(val) {
          return that.oldCheckedItemIds.indexOf(val) < 0; // 找不到
        });
      } else if (newVal.length < this.oldCheckedItemIds.length) { // 减少
        selected = false;
        diffItemIds = [...this.oldCheckedItemIds].filter(function(val) {
          return newVal.indexOf(val) < 0; // 找不到
        });
      } else {
        return;
      }
      updateCartSelected(diffItemIds, selected).then(data => {
        this.handleData(data);
      })
      // debugger;
    },
    onSelectAll(newVal) {
      if (this.checkedAll === undefined) {
        return;
      }
      // debugger;
      // updateCartSelected(this.getItemIds(), newVal).then(data => {
      //   this.handleData(data);
      // })
      if (newVal) {
        this.onItemSelectedChange(this.getItemIds());
      } else {
        // alert('有 bug ，后续修复');
        // this.onItemSelectedChange(this.getItemIds());
        // TODO 芋艿，暂时有 bug 。后续修复
      }
    },
    onSubmit() {
      this.$router.push('/order?from=cart')
    },
    convertProduct(item) {
      // debugger;
      return {
        ...item.spu,
        quantity: item.buyQuantity,
        price: item.buyPrice || item.price,
        sku: {
          ...item,
          spu: undefined,
        },
        selected: item.selected,
      };
    }
  },
  mounted() {
    // 获得购物车列表
    listCart().then(data => {
      this.handleData(data);
    });
  }
};
</script>

<style lang="less">
.card-goods {
  font-size: 12px;
  &__item {
    position: relative;
    .van-checkbox{
      width: 20px;
      height: 20px;
      top: 40px;
      left: 5px;
      z-index: 1;
      position: absolute;
    }
    .additional{
      width: 100%;
        padding-left: 15px;
    box-sizing: border-box;
    }
  }
}
.head{
      padding-left: 5px;
  border-bottom: 1px solid #eee;
}
.card{
  background: #f7f7f7;
  .van-submit-bar__bar {
      border-top: 1px solid #f7f7f7;
      .van-checkbox{
        padding-left: 5px;
      }
  }
  .promotion{
      .van-tag {
          line-height: 12px;
          margin-right: 5px;
      }
      .van-cell__title{

      flex: 4;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
      }
    }
    .promotion-group{
      margin-top: 10px;
      box-shadow: 5px 5px 5px #e5e5e5;
    }
}


</style>
