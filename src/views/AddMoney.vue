<template>
  <div id="view-addMoney">
    <div class="addMoney-header">
      <van-tabs color="#f6717d"
                v-model="type">
        <van-tab title="支出"></van-tab>
        <van-tab title="收入"></van-tab>
      </van-tabs>
    </div>
    <div class="addMoney-val">
      <div class="addMoney-val-account"
           @click="isShowAccount = true">
        <div class="account-name">{{selectedAccount.name}}</div>
      </div>
      <div class="addMoney-val-money"
           @click="isShowNumKey = !isShowNumKey">￥{{value}}</div>
    </div>
    <div class="addMoney-main">
      <van-swipe :autoplay="0"
                 indicator-color="#f6717d"
                 :loop="false">
        <van-swipe-item v-for="(page, pageIdx) in [outCategoryList,inCategoryList][type]"
                        :key="pageIdx">
          <div class="addMoney-category">
            <type-icon class="addMoney-category-item"
                       checker
                       v-for="(category) in page"
                       :key="category._id"
                       :_id="category._id"
                       :type="Number(category.type)"
                       :icon="category.icon"
                       :title="category.name"
                       title-position="bottom"
                       :selected="selectedCategory == category._id"
                       @select="handleSelectCategory" />
            <type-icon 
              class="addMoney-category_setting"
              icon="shezhixuanzhong"
              title-position="bottom"
              title="自定义"
              v-if="pageIdx === [outCategoryList,inCategoryList][type].length - 1"
              @select="$router.push('/categoryManage')" />
          </div>
        </van-swipe-item>
      </van-swipe>
      <van-cell-group>
        <van-cell title="时间"
                  is-link
                  :value="$moment(date).format('YYYY-MM-DD HH:mm')"
                  @click="isShowDate = !isShowDate" />
        <van-field v-model="note"
                   placeholder="输入备注" />
      </van-cell-group>
    </div>
    <div class="addMoney-footer">
      <van-button type="primary"
                  size="large"
                  plain
                  v-if="!moneyId"
                  @click="handleCreateMore">再记一笔</van-button>
      <van-button type="primary"
                  size="large"
                  @click="handleSave">保存</van-button>
    </div>
    <van-number-keyboard :show="isShowNumKey"
                         extra-key="."
                         @blur="isShowNumKey = false"
                         theme="custom"
                         @input="handleInputNumber"
                         @delete="handleDeleteNumber"
                         close-button-text="确认" />
    <van-popup v-model="isShowDate"
               position="bottom">
      <van-datetime-picker v-model="modalDate"
                           @confirm="confirmDate"
                           @cancel="cancelDate"
                           type="datetime" />
    </van-popup>
    <van-popup v-model="isShowAccount"
               position="bottom">
      <van-radio-group v-model="selectedAccountId">
        <van-cell-group>
          <van-cell v-for="item in accountList"
                    :key="item._id"
                    :title="item.name"
                    :label="item.summary"
                    size="large"
                    clickable
                    @click="handleSelectAccount(item._id)">
            <van-radio 
              :name="item._id" 
              checked-color="#f6717d"
            />
          </van-cell>
        </van-cell-group>
      </van-radio-group>
    </van-popup>
  </div>
</template>

<script>
import {
  addMoney,
  updateMoney,
  getCategoryList,
  getAccountList,
  getMoneyDetail,
  getMoneySum,
  getBudget
} from "../api/api.js";
import { Notify } from "vant";
const CATEGORY_PAGE_SIZE = 15; // 每页分类数
const MAX_MONEY = 9; // 金额位数
const CLIENT_WIDTH = document.body.clientWidth;
const TODAY = new Date();

export default {
  name: "addMoney",
  data() {
    return {
      intStack: [],
      floatStack: [],
      value: "0",
      isInputInt: true,
      type: 0,
      note: "",
      inCategoryList: [],
      outCategoryList: [],
      accountList: [],
      selectedCategory: 0,
      selectedAccountId: "",
      tabWidth: CLIENT_WIDTH / 2,
      isShowNumKey: false,
      isShowDate: false,
      isShowAccount: false,
      date: TODAY,
      modalDate: TODAY,
      today: TODAY,
      moneyId: "",
      outcome: 0,
      budget: 0
    };
  },
  created() {
    this.moneyId = this.$route.query.moneyId || "";
    this.date = this.$route.query.date
      ? new Date(Number(this.$route.query.date))
      : TODAY;
    this.modalDate = this.date;
    this.init();
    this.getValue();
  },
  watch: {
    type(newVal) {
      this.selectFirstCategory();
    }
  },
  computed: {
    selectedAccount() {
      return this.accountList.find(item => item._id == this.selectedAccountId);
    }
  },
  methods: {
    init() {
      getCategoryList().then(res => {
        const inCategorys = [];
        const outCategorys = [];
        res.data.list.forEach(elem => {
          if (elem.status === 1) {
            if (elem.type == 0) {
              outCategorys.push(elem);
            } else if (elem.type == 1) {
              inCategorys.push(elem);
            }
          }
        });
        this.inCategoryList = this.groupToPage(inCategorys, CATEGORY_PAGE_SIZE);
        this.outCategoryList = this.groupToPage(
          outCategorys,
          CATEGORY_PAGE_SIZE
        );
        if (!this.moneyId) this.selectFirstCategory();
      });
      getAccountList().then(res => {
        this.accountList = res.data.list;
        this.selectedAccountId = this.accountList[0]._id;
      });
      if (this.moneyId) {
        getMoneyDetail({ moneyId: this.moneyId }).then(res => {
          const money = res.data.detail;
          this.value = money.value.toString();
          this.date = money.moneyTime;
          this.note = money.note;
          this.selectedCategory = res.data.detail.categoryId._id;
          this.selectedAccountId = res.data.detail.accountId._id;

          const { intStack, floatStack } = this.numToStack(this.value);
          this.intStack = intStack;
          this.floatStack = floatStack;
        });
      }
      const moneyTimeStart = this.$moment().startOf("month");
      const sumData = {
        searchValue: {
          moneyTimeStart: moneyTimeStart.format("YYYY-MM-DD"),
          moneyTimeEnd: moneyTimeStart.add(1, "month").format("YYYY-MM-DD")
        }
      };
      getMoneySum(sumData).then(res => {
        const sums = res.data.result;
        sums.find(item => {
          if (item._id.type === 0) {
            this.outcome = item.value;
            return;
          }
        });
      });
      getBudget().then(res => {
        this.budget = res.data.value;
      });
    },
    groupToPage(list, pageSize) {
      let result = [];
      for (let x = 0; x < Math.ceil(list.length / pageSize); x++) {
        let start = x * pageSize;
        let end = start + pageSize;
        result.push(list.slice(start, end));
      }
      return result;
    },
    selectFirstCategory() {
      this.selectedCategory = [this.outCategoryList, this.inCategoryList][
        this.type
      ][0][0]._id;
    },

    handleInputNumber(key) {
      if (this.value.length < MAX_MONEY || this.value == "0") {
        if (key == ".") this.isInputInt = false;
        else {
          if (this.isInputInt) {
            this.intStack.push(key);
          } else {
            if (this.floatStack.length <= 1) this.floatStack.push(key);
          }
        }
      }
      this.getValue();
    },
    handleDeleteNumber() {
      const hasInt = Boolean(this.intStack.length),
        hasFloat = Boolean(this.floatStack.length);
      if (hasFloat) {
        this.floatStack.pop();
        if (hasFloat) {
          this.isInputInt = true;
        }
      } else {
        if (hasInt) {
          this.intStack.pop();
        }
      }
      this.getValue();
    },
    getValue() {
      const hasInt = Boolean(this.intStack.length),
        hasFloat = Boolean(this.floatStack.length);
      if (hasInt || hasFloat) {
        if (!hasInt) {
          return (this.value = `0.${this.floatStack.join("")}`);
        } else if (!hasFloat) {
          return (this.value = `${this.intStack.join("")}`);
        }
        return (this.value = `${this.intStack.join("")}.${this.floatStack.join(
          ""
        )}`);
      } else return (this.value = "0");
    },
    numToStack(num) {
      const nums = typeof num == "number" ? num.toString : num;
      const intFloat = nums.split(".");
      let intStack = [],
        floatStack = [];

      intStack = intFloat[0].split("");
      if (intFloat.length > 1) {
        floatStack = intFloat[1].split("");
      }
      return { intStack, floatStack };
    },
    saveMoney() {
      const data = {
        type: this.type,
        value: this.value,
        categoryId: this.selectedCategory,
        accountId: this.selectedAccountId,
        moneyTime: this.$moment(this.date).format("YYYY-MM-DD HH:mm:ss"),
        note: this.note
      };
      if (this.value == "0") {
        this.$notify({
          message: "请填写金额",
          background: this.$color["error"]
        });
        return;
      }
      const option = { successDialog: true, goBack: true };
      this.checkBudget();
      const type = this.type;
      if (this.moneyId) {
        updateMoney({ ...data, moneyId: this.moneyId }, option).then(res => {
          // this.$router.go(-2);
        });
      } else {
        addMoney(data, option).then(res => {
          this.type = type
          // this.$router.go(-2);
        });
      }
    },
    handleSelectCategory(id) {
      this.selectedCategory = id;
    },
    handleSelectAccount(data) {
      this.selectedAccountId = data;
      setTimeout(() => {
        this.isShowAccount = false;
      }, 200);
    },
    confirmDate() {
      this.date = this.modalDate;
      this.isShowDate = false;
    },
    cancelDate() {
      this.modalDate = this.date;
      this.isShowDate = false;
    },
    handleSave() {
      this.saveMoney();
      // this.$router.push(-1);
    },
    handleCreateMore() {
      this.saveMoney();
      this.$router.push("/addMoney");
    },
    async checkBudget(fn) {
      if (
        this.type === 0 &&
        this.outcome < this.budget &&
        this.outcome + Number(this.value) > this.budget
      ) {
        this.type = 0;
        await this.$dialog.alert({
          title: "预算超额提醒",
          message: "您的本月预算已超额，请注意控制支出"
        });
      }
    },
    goBack() {
      console.log("this.$router.go: ", this.$router.go);
    }
  }
};
</script>

<style lang="less">
@import "../assets/variable.less";
#view-addMoney {
  display: flex;
  flex-direction: column;
  height: 100%;
  color: @primaryTextColor;
  background-color: #fff;
  .van-popup {
    margin-bottom: 12px;
    width: 90%;
    border-radius: 6px;
    .van-cell--large {
      padding-top: 20px;
      padding-bottom: 20px;
    }
  }
}
.addMoney-header {
  border-bottom: 1px solid @dividerColor;
}
.addMoney-val {
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 72px;
  border-bottom: 1px solid @dividerColor;
  &-account {
    display: flex;
    align-items: center;
    padding: 0 20px;
    height: 60%;
    border-right: 1px solid @dividerColor;
    font-size: 18px;
    .account-icon {
      display: flex;
      justify-content: center;
      align-items: center;
      width: 36px;
      height: 36px;
      border-radius: 4px;
      background: #2873ff;
      color: #fff;
      font-size: 40px;
    }
  }
  &-money {
    flex: 1;
    padding: 0 12px;
    height: 100%;
    line-height: 72px;
    text-align: right;
    font-size: 30px;
  }
}

.addMoney-main {
  flex: 1;
}

.addMoney-footer {
  display: flex;
  padding: 12px 10px;
  .van-button:not(:last-child) {
    margin-right: 8px;
  }
}

.van-swipe {
  .addMoney-category {
    display: flex;
    flex-wrap: wrap;
    justify-content: flex-start;
    padding: 14px 18px 12px;
    height: 220px;
    &-item,
    &_setting {
      width: 20%;
      margin: 4px 0;
    }
    &_setting .typeIcon-icon {
      border: 1px solid #cccccc;
      background-color: #fff;
      color: #cccccc;
    }
  }
}

.addMoney-infoBar {
  display: flex;
  align-items: center;
  background: #fff;
  border-top: 1px solid @dividerColor;
  .addMoney-infoBar-date {
    display: inline-block;
    padding: 0 10px;
    line-height: 36px;
    border-right: 1px solid @dividerColor;
    .iconfont {
      font-size: 30px;
    }
  }
}

.addMoney-tagBar {
  display: flex;
  align-items: center;
  height: 32px;
  width: 100%;
  border-top: 1px solid @dividerColor;
}
</style>