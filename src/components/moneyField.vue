<!--  -->
<template>
  <div>
    <van-field
      v-model="value"
      :label="label"
      @click="handleValue"
      readonly
      input-align="right"
      :placeholder="placeholder"
    />
    <van-number-keyboard 
      :show="isShowNumKey"
      extra-key="."
      @blur="isShowNumKey = false"
      theme="custom"
      @input="handleInputNumber"
      @delete="handleDeleteNumber"
      close-button-text="确认" />
  </div>
</template>

<script>
const MAX_MONEY = 9; // 金额位数
export default {
  props: {
    value: {
      type: Number,
      default: 0
    },
    label: {
      type: String
    },
    placeholder: {
      type: String
    }
  },
  data() {
    return {
      isShowNumKey: false,
      intStack: [],
      floatStack: [],
      isInputInt: true,
      moneyValue: 0
    };
  },

  computed: {},

  watch: {
    value(newVal) {
      this.moneyValue = newVal;
      const nums = newVal.toString();
    const intFloat = nums.split(".");
    this.intStack = intFloat[0].split("");
    if (intFloat.length > 1) {
      this.floatStack = intFloat[1].split("");
    }
    }
  },

  model: {
    prop: "value",
    event: "change"
  },

  created() {
    this.moneyValue = this.value;
    const nums = this.value.toString();
    const intFloat = nums.split(".");
    this.intStack = intFloat[0].split("");
    if (intFloat.length > 1) {
      this.floatStack = intFloat[1].split("");
    }
  },

  methods: {
    handleValue() {
      this.isShowNumKey = true;
    },
    handleInputNumber(key) {
      if (
        this.moneyValue.toString().length < MAX_MONEY ||
        this.moneyValue == 0
      ) {
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
      this.$emit("change", this.moneyValue);
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
      if(this.intStack.length == 1 && this.intStack[0] == '-') {
        this.intStack.pop();
      }
      this.getValue();
      this.$emit("change", this.moneyValue);
    },
    getValue() {
      const hasInt = Boolean(this.intStack.length),
        hasFloat = Boolean(this.floatStack.length);
      if (hasInt || hasFloat) {
        if (!hasInt) {
          return (this.moneyValue = Number(`0.${this.floatStack.join("")}`));
        } else if (!hasFloat) {
          return (this.moneyValue = Number(`${this.intStack.join("")}`));
        }
        return (this.moneyValue = Number(
          `${this.intStack.join("")}.${this.floatStack.join("")}`
        ));
      } else return (this.moneyValue = 0);
    },
    numToStack(num) {
      const nums = num.toString();
      const intFloat = nums.split(".");
      let intStack = [],
        floatStack = [];

      intStack = intFloat[0].split("");
      if (intFloat.length > 1) {
        floatStack = intFloat[1].split("");
      }
      return { intStack, floatStack };
    }
  }
};
</script>

<style lang='less' scoped>
</style>