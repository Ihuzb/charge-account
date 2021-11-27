<template>
  <div id="app">
    <el-row :gutter="20">
      <el-row class="topRow">
        <el-col :span="6">
          <el-select v-model="selectValue" clearable placeholder="请选择账单时间">
            <el-option v-for="item in timeFilter" :key="item" :value="item"></el-option>
          </el-select>
        </el-col>
        <el-col :span="6">
          <el-select v-model="selectCategorValue" clearable placeholder="请选择账单分类">
            <el-option v-for="item in categoryFilter" :key="item" :value="item"
                       :label="item|categoryFor(that)"></el-option>
          </el-select>
        </el-col>
        <el-col :span="6">
          <el-button v-if="!isSetId" type="primary" @click="addInfo">添加账单</el-button>
          <div v-else>
            <el-button type="primary" @click="saveInfo">保存</el-button>
            <el-button @click="removeInfo">取消</el-button>
          </div>
        </el-col>
      </el-row>
      <el-col :span="14">
        <el-table border row-key="id" :data="billList|billShow" height="500">
          <el-table-column
              fixed
              prop="timeFor"
              sortable
              label="账单时间">
          </el-table-column>
          <el-table-column
              prop="type"
              sortable
              label="账单类型">
            <template slot-scope="scope">
              <p>{{ scope.row.type|typeFor(that) }}</p>
            </template>
          </el-table-column>
          <el-table-column
              prop="category"
              sortable
              label="账单分类">
            <template slot-scope="scope">
              <p v-if="isSetId!=scope.row.id">{{ scope.row.category|categoryFor(that) }}</p>
              <el-select v-else size="mini" v-model="addBillInfo.category" clearable>
                <el-option v-for="item in Object.values(categoryKey)" :key="item.id" :label="item.name"
                           :value="item.id"></el-option>
              </el-select>
            </template>
          </el-table-column>
          <el-table-column
              prop="amount"
              sortable
              :sort-method="sortMethod"
              label="账单金额">
            <template slot-scope="scope">
              <p v-if="isSetId!=scope.row.id">{{ scope.row.amount|amountFor }}</p>
              <el-input v-else v-model="addBillInfo.amount" size="mini" type="number"/>
            </template>
          </el-table-column>
        </el-table>
      </el-col>
      <el-col :span="10">
        <el-table
            border
            ref="table"
            :data="sumList.data"
            row-key="category"
            :summary-method="summaries"
            :span-method="spanMethod"
            show-summary
            height="500">
          <el-table-column
              prop="type"
              label="账单类型">
            <template slot-scope="scope">
              <p>{{ scope.row.type|typeFor(that,"amount") }}</p>
            </template>
          </el-table-column>
          <el-table-column
              prop="category"
              label="账单分类">
            <template slot-scope="scope">
              <p>{{ scope.row.category|categoryFor(that) }}</p>
            </template>
          </el-table-column>
          <el-table-column
              prop="amount"
              label="账单金额">
            <template slot-scope="scope">
              <p>{{ scope.row.amount|amountFor }}</p>
            </template>
          </el-table-column>
        </el-table>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import {csvParse} from "d3-dsv"
import moment from "moment"
// 自定义字段维护
const keyValue = {isShow: "isShow", id: "id", timeFor: "timeFor"}
export default {
  name: 'App',
  data() {
    return {
      billList: [],
      sumList: {rowNum: {}, data: [], amountNum: {}},
      that: this,// 定义this指向
      categoriesList: [],
      typeKey: {"1": "收入", "0": "支出"},
      categoryKey: {},// 分类map结构
      timeFilter: [],// 日期过滤 月份条件
      categoryFilter: [],// 分类过滤 分类条件
      selectValue: "",// 月份筛选条件
      selectCategorValue: "",// 分类筛选条件
      addBillInfo: {},
      isSetId: "",// 处在编辑模式的id
    }
  },
  watch: {
    addBillInfo: {
      handler: function (newVal) {
        if (newVal.category) {
          this.addBillInfo.type = this.categoryKey[newVal.category].type
        }
      },
      deep: true
    },
    selectValue: function () {
      this.selectInfo();
    },
    selectCategorValue: function () {
      this.selectInfo();
    },
    billList: {
      handler: function (newVal) {
        this.watchBillList(newVal)
      },
      deep: true
    }
  },
  created() {
    this.getData();
  },
  methods: {
    // billList监听到的修改需要执行的操作
    watchBillList: function (newVal) {
      let info = {}, sumList = [], rowNum = {}, amountNum = {};
      // 根据账单类型和账单分类进行统计
      newVal.filter(v => (v.isShow == void 0 || v.isShow) && v.category && v.amount).forEach(v => {
        if (!info[v.type]) {
          info[v.type] = {}
          info[v.type][v.category] = Number(v.amount)
          rowNum[v.type] = 1
        } else {
          if (!info[v.type][v.category]) {
            info[v.type][v.category] = Number(v.amount)
            rowNum[v.type] += 1
          } else {
            info[v.type][v.category] += Number(v.amount)
          }
        }
      });
      for (let type in info) {
        // 求出所有支出和收入的合计 并以类型为key进行保存
        let allTypeMoney = Object.values(info[type]).reduce((num, total) => num + total, 0);
        amountNum[type] = allTypeMoney
        // 单元格合并记录 支出和收入开始行数 各占多少行
        rowNum[type] = [sumList.length, rowNum[type]]
        for (let category in info[type]) {
          sumList.push({type, category, amount: info[type][category]})
        }
      }
      sumList = sumList.sort((a, b) => {
        if (a.type != b.type) {
          return a.type - b.type
        } else {
          return b.amount - a.amount
        }
      })
      this.$set(this.sumList, "rowNum", rowNum);
      this.$set(this.sumList, "data", sumList);
      this.$set(this.sumList, "amountNum", amountNum);
    },
    // 统一筛选方法
    selectInfo: function () {
      let categorValue = this.selectCategorValue;
      let timeValue = this.selectValue;
      this.billList = this.billList.map(v => {
        let time = v.timeFor.split("-")[1];
        return Object.assign(v, {
          [keyValue["isShow"]]: (timeValue ? (time == timeValue) : true) && (categorValue ? categorValue == v.category : true)
        })
      });
    },
    // 总金额合计行展示
    summaries: function (param) {
      const {columns, data} = param;
      const sums = [];
      if (data.length) {
        columns.forEach((column, index) => {
          if (index === 0) {
            sums[index] = '总价';
          }
          if (index == 2) {
            sums[1] = "N/A"
            sums[index] = (data.reduce((num, total) => {
              return num + Number(total.amount)
            }, 0)).toFixed(2);
          }
        })
        this.$nextTick(() => {
          this.$refs.table.doLayout();
        });
      }
      return sums;
    },
    // 金额排序
    sortMethod: function (a, b) {
      return a.amount - b.amount
    },
    // 合并单元格
    spanMethod: function ({row, rowIndex, columnIndex}) {
      let sumList = this.sumList.rowNum;
      if (columnIndex == 0) {
        if (sumList[row.type][0] == rowIndex) {
          return {
            rowspan: sumList[row.type][1],
            colspan: 1
          };
        } else {
          return {
            rowspan: 0,
            colspan: 0
          };
        }
      }
    },
    // 保存账单
    saveInfo: function () {
      this.isSetId = ""
    },
    // 取消账单
    removeInfo: function () {
      this.billList.shift();
      this.isSetId = ""
    },
    // 添加账单
    addInfo: function () {
      let billList = this.billList, info = {};
      for (let key in billList[0]) {
        let value = ""
        if (key == "id") {
          value = `a${billList.length}`
          this.isSetId = value;
        } else if (key == "time") {
          value = moment().valueOf()
        } else if (key == "timeFor") {
          value = moment().format("YYYY-MM-DD")
        }
        info[key] = value
      }
      this.selectValue = ""
      this.selectCategorValue = ""
      this.addBillInfo = info;
      this.billList.unshift(this.addBillInfo);
    },
    // 读取数据
    getData: function () {
      // 同时获取账单数据和分类数据
      Promise.all([
        this.$axios.get("./data/bill.csv"),
        this.$axios.get("./data/categories.csv"),
      ]).then(res => {
        let categoryKey = {}, timeFilter = [], categoryFilter = [];
        let [bill, categories] = res;
        // 账单数据
        this.billList = csvParse(bill.data).map(function (v, i) {
          // 时间戳转换用于展示
          let timeFor = moment(parseInt(v.time)).format("YYYY-MM-DD");
          // 记录所有月份筛选用
          timeFilter.push(timeFor.split("-")[1]);
          categoryFilter.push(v.category);
          // 创建id，展示和筛选用时间字段
          return Object.assign(v, {[keyValue["id"]]: `a${i}`, [keyValue["timeFor"]]: timeFor})
        });
        // 保存月份筛选条件
        this.timeFilter = [...new Set(timeFilter)];
        // 保存分类筛选条件
        this.categoryFilter = [...new Set(categoryFilter)];

        // 保存分类数据
        let categoriesList = Object.freeze(csvParse(categories.data));
        this.categoriesList = categoriesList;
        // 处理分类id对应的内容
        categoriesList.forEach(v => {
          categoryKey[v.id] = v
        });
        this.categoryKey = Object.freeze(categoryKey);
      });
    },
  },
  filters: {
    // 金额格式化
    amountFor: function (value) {
      return value ? Number(value).toFixed(2) : value;
    },
    // 类型格式化
    typeFor: function (value, that, type) {
      // 特殊情况 仅在需要单独展示 支出或收入的总金额
      if (type == "amount") {
        return value ? `${that.typeKey[value]}  ${that.sumList.amountNum[value].toFixed(2)}` : value
      } else {
        return value ? that.typeKey[value] : value
      }

    },
    // 分类格式化
    categoryFor: function (value, that) {
      return value ? that.categoryKey[value].name : value
    },
    // 账单展示过滤
    billShow: function (value) {
      // isShow true 展示  false 不展示
      if (value.length) {
        return value.filter(v => v.isShow == void 0 || v.isShow)
      } else {
        return value;
      }
    }
  }
}
</script>

<style scoped>
#app {
  padding: 10px;
}

.topRow {
  margin-bottom: 10px;
}

</style>
