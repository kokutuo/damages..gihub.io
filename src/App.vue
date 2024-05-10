<template>
  <div id="app">
    <div class="left">
      <el-form :model="formData" label-width="100px">
        <el-form-item label="本金">
          <el-input v-model="formData.total"></el-input>
        </el-form-item>
        <el-form-item label="利息起算日期">
          <el-date-picker
            type="date"
            placeholder="选择日期"
            style="width: 100%"
            v-model="formData.startDate"
            value-format="yyyy-MM-dd"
          ></el-date-picker>
        </el-form-item>
        <el-form-item label="利息终止日期">
          <el-date-picker
            type="date"
            placeholder="选择日期"
            style="width: 100%"
            v-model="formData.endDate"
            value-format="yyyy-MM-dd"
          ></el-date-picker>
        </el-form-item>
        <el-form-item label="lpr倍率">
          <el-input v-model="formData.rate"></el-input>
        </el-form-item>
        <el-form-item label="lpr">
          <el-select v-model="formData.lprY" placeholder="请选择活动区域">
            <el-option label="一年期" value="oneY"></el-option>
            <el-option label="五年期" value="fiveY"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="onSubmit">计算</el-button>
          <el-button type="primary" @click="exportExcel">导出</el-button>
        </el-form-item>
      </el-form>
    </div>
    <div class="right">
      <el-table
        :summary-method="getSummaries"
        :data="tableData"
        show-summary
        style="width: 100%"
      >
        <el-table-column prop="total" label="本金"> </el-table-column>
        <el-table-column prop="start" label="利息起算日期"> </el-table-column>
        <el-table-column prop="end" label="利息截止日期"> </el-table-column>
        <el-table-column prop="days" label="计算天数"> </el-table-column>
        <el-table-column prop="lpr" label="LPR"> </el-table-column>
        <el-table-column prop="interest" label="利息"> </el-table-column>
      </el-table>
    </div>
  </div>
</template>

<script>
/* eslint-disable */
import Excel from "./utils/excel";
import Dayjs from "dayjs";

export default {
  name: "App",
  data: function () {
    return {
      visible: false,
      formData: {
        total: null,
        startDate: "",
        endDate: "",
        rate: 1,
        lprY: "oneY",
        lprArr: [],
      },
      tableData: [],
    };
  },
  mounted() {
    this.lprArr = require("./assets/lpr.json");
    var isSameOrBefore = require("dayjs/plugin/isSameOrBefore");
    Dayjs.extend(isSameOrBefore);
    var isSameOrAfter = require("dayjs/plugin/isSameOrAfter");
    Dayjs.extend(isSameOrAfter);
  },
  methods: {
    onSubmit() {
      let res = [];
      let validLpr = this.lprArr.filter((item) => {
        return (
          Dayjs(item.date).isSameOrBefore(Dayjs(this.formData.endDate)) &&
          Dayjs(item.date).isSameOrAfter(
            Dayjs(this.formData.startDate).subtract(30, "day")
          )
        );
      });
      for (let i = validLpr.length - 1; i >= 0; i--) {
        const element = validLpr[i];

        let days = 0, //间隔天数
          start = "", // 起算日期
          end = "", //截至日期
          lpr = "", //lpr
          interest = ""; //利息

        if (i === validLpr.length - 1) {
          days = this.diff(validLpr[i - 1].date, this.formData.startDate);
          start = this.formData.startDate;
          end = Dayjs(validLpr[i - 1].date)
            .subtract(1, "day")
            .format("YYYY-MM-DD");
        } else if (i === 0) {
          days = this.diff(this.formData.endDate, element.date) + 1;
          start = element.date;
          end = this.formData.endDate;
        } else {
          days = this.diff(validLpr[i - 1].date, element.date);
          start = element.date;
          end = Dayjs(validLpr[i - 1].date)
            .subtract(1, "day")
            .format("YYYY-MM-DD");
        }

        lpr = element[this.formData.lprY];

        interest =
          (Number(this.formData.total) *
            Number(days) *
            Number(lpr) *
            Number(this.formData.rate)) /
          365 /
          100;

        res.push({
          total: this.formData.total,
          start,
          end,
          days,
          lpr,
          interest,
        });
      }
      console.log(
        "🚀 ~ file: App.vue:86 ~ onSubmit ~ res:",
        JSON.stringify(res)
      );
      this.tableData = this.mergeLPR(res);
    },

    exportExcel() {
      let data = [...this.tableData];

      // data.push({
      //   total: "合计",
      //   start: "",
      //   end: "",
      //   days: "",
      //   lpr: "",
      //   interest: this.round(
      //     data.reduce((prev, curr) => {
      //       const value = Number(curr.interest);
      //       if (!isNaN(value)) {
      //         return prev.interest + curr.interest;
      //       } else {
      //         return prev.interest;
      //       }
      //     }, 0)
      //   ),
      // });

      console.log("🚀 ~ file: App.vue:195 ~ exportExcel ~ data:", data);

      let header = [
        "本金",
        "利息起算日期",
        "利息截止日期",
        "计算天数",
        "LPR",
        "利息",
      ];
      Excel.exportExcel(
        this.tableData,
        "违约金计算表格" + Dayjs().format() + ".xlsx",
        header
      );
    },

    getSummaries(param) {
      const { columns, data } = param;
      const sums = [];
      columns.forEach((column, index) => {
        if (index === 0) {
          sums[index] = "合计";
          return;
        }
        const values = data.map((item) => Number(item[column.property]));
        if (!values.every((value) => isNaN(value))) {
          sums[index] = this.round(
            values.reduce((prev, curr) => {
              const value = Number(curr);
              if (!isNaN(value)) {
                return prev + curr;
              } else {
                return prev;
              }
            }, 0)
          );
        } else {
          sums[index] = "N/A";
        }
      });

      return sums;
    },

    round(x) {
      return (Math.round(x * (10 ^ 2)) / (10 ^ 2)).toFixed(2);
    },

    diff(date1, date2) {
      return Dayjs(date1).diff(date2, "day");
    },

    mergeLPR(arr) {
      if (arr.length === 0) return arr;

      let result = [arr[0]];
      for (let i = 1; i < arr.length; i++) {
        if (arr[i].lpr === result[result.length - 1].lpr) {
          result[result.length - 1].end = arr[i].end;
          result[result.length - 1].days += arr[i].days;
          result[result.length - 1].interest += arr[i].interest;
        } else {
          result.push(arr[i]);
        }
      }
      return result;
    },
  },
};
</script>

<style>
body {
  margin: 0;
  padding: 10px;
  box-sizing: border-box;
  background: #eeeeee9d;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  display: flex;
}

.left,
.right {
  box-sizing: border-box;
  background: #ffffff;
  border-radius: 10px;
  padding: 10px;
  height: calc(100vh - 20px);
  overflow: auto;
}

.left {
  box-sizing: border-box;
  width: 600px;
  margin-right: 20px;
}

.right {
  flex-grow: 1;
}
</style>
