<!--  -->
<template>
  <div class="content">
    <div class="left">
      <div class="left-title">充电站列表</div>
      <div class="list">
        <div class="list-input">
          <el-input placeholder="输入关键字进行搜索" v-model="filterText">
          </el-input>
        </div>
        <div class="tree">
          <el-tree
            class="filter-tree"
            :data="dataList"
            :props="defaultProps"
            :default-checked-keys="defaultkeys"
            default-expand-all
            node-key="stationId"
            highlight-current
            @node-click="handleNodeClick"
            :filter-node-method="filterNode"
            ref="tree"
          >
          </el-tree>
        </div>
      </div>
    </div>
    <div class="right" v-loading="loading">
      <div class="right-title">实时监控</div>
      <div class="name" v-if="stationName">{{ stationName }}</div>
      <el-empty
        description="暂无数据"
        v-if="chargingArr.length === 0"
      ></el-empty>
      <div class="list" v-if="chargingArr.length">
        <div class="station" v-for="item in chargingArr" :key="item.pileId">
          <div class="info">
            <div class="num">
              <img :src="item.gunStatus === 1 ? online : offline" />
              <span>{{ item.pileId || "--" }}-{{ item.deviceId }}</span>
            </div>
            <div class="status">
              {{ getStatus(item.gunStatus) }}
            </div>
          </div>
          <div class="mess">
            <el-popover
              placement="right"
              title="模拟充电桩"
              width="200"
              trigger="click"
            >
              <div class="operate">
                <el-button type="primary" @click="operateHandler('start', item)"
                  >启动模拟充电桩</el-button
                >
                <el-button
                  type="primary"
                  v-if="item.gunStatus != 1"
                  @click="operateHandler('link', item)"
                  >插枪</el-button
                >
                <el-button
                  type="primary"
                  v-if="item.gunStatus != 1"
                  @click="operateHandler('startCharge', item)"
                  >开始充电</el-button
                >
                <el-button
                  type="primary"
                  @click="operateHandler('endCharge', item)"
                  >停止充电</el-button
                >
                <el-button
                  type="primary"
                  v-if="item.gunInsert == 1"
                  @click="operateHandler('unlink', item)"
                  >拔枪</el-button
                >

                <el-button type="primary" @click="operateHandler('stop', item)"
                  >停止模拟充电桩</el-button
                >
              </div>

              <!-- <el-button slot="reference" class="btn">click 激活</el-button> -->
              <i class="el-icon-s-tools operateBtn" slot="reference"></i>
            </el-popover>

            <div class="item">
              <div class="item-value">
                {{
                  item.chargingCurrent
                    ? Number(item.chargingCurrent).toFixed(2)
                    : 0
                }}A
              </div>
              <div class="item-name">电流</div>
            </div>
            <div class="item">
              <div class="item-value">
                {{
                  item.chargingCdgl && item.chargingCurrent
                    ? (
                        Number(item.chargingCdgl) / Number(item.chargingCurrent)
                      ).toFixed(2)
                    : 0
                }}
              </div>
              <div class="item-name">电压</div>
            </div>
            <div class="item">
              <div class="item-value">
                {{
                  item.chargingCdgl ? Number(item.chargingCdgl).toFixed(2) : 0
                }}KW
              </div>
              <div class="item-name">功率</div>
            </div>
            <div class="item">
              <div class="item-value">{{ item.useCount || 0 }}</div>
              <div class="item-name">次数</div>
            </div>
            <div class="item">
              <div class="item-value">
                {{ item.realHour ? (item.realHour * 60).toFixed(2) : 0 }}分钟
              </div>
              <div class="item-name">时间</div>
            </div>
            <div class="item">
              <div class="item-value">
                {{ item.consumePower ? item.consumePower.toFixed(2) : 0 }}度
              </div>
              <div class="item-name">电量</div>
            </div>
          </div>
        </div>
      </div>

      <pagination
        v-show="total > 0"
        :total="total"
        :page.sync="queryParams.pageNum"
        :limit.sync="queryParams.pageSize"
        @pagination="getNewStatus"
      />
    </div>
  </div>
</template>
·
<script>
import { listStation } from "@/api/operator/station";
import {
  listPort,
  startCharge,
  unlink,
  stop,
  start,
  link,
  endCharge,
} from "@/api/operator/port";
export default {
  components: {},
  data() {
    return {
      offline: require("@/assets/images/offline.png"),
      online: require("@/assets/images/online.png"),
      stationName: "",
      filterText: "",
      dataList: [],
      // 总条数
      total: 0,
      defaultkeys: [],
      defaultProps: {
        label: "stationName",
      },
      queryParams: {
        pageNum: 1,
        pageSize: 10,
      },
      loading: false,
      chargingArr: [],
      timer: null,
      stationId: "",
    };
  },
  computed: {},
  watch: {
    filterText(val) {
      this.$refs.tree.filter(val);
    },
  },
  methods: {
    /** 查询站点列表 */
    getList() {
      let tenantId = this.$store.state.user.tenantid;
      this.loading = true;
      listStation({
        tenantId,
        pageNum: 1,
        pageSize: 100000,
      }).then((response) => {
        this.dataList = response.data;
        if (this.dataList.length) {
          this.defaultkeys = [this.dataList[0].stationId];
          this.stationName = this.dataList[0].stationName;
          this.$nextTick(() => {
            this.$refs.tree.setCurrentKey(this.defaultkeys[0]);
            this.handleNodeClick(this.dataList[0]);
          });
        }

        this.loading = false;
      });
    },
    filterNode(value, data) {
      if (!value) return true;
      return data.stationName.indexOf(value) !== -1;
    },
    handleNodeClick(data) {
      if (!data.stationId) {
        return;
      }
      this.loading = true;
      clearInterval(this.timer);
      this.stationName = data.stationName;
      this.stationId = data.stationId;
      this.queryParams.pageNum = 1;
      listPort({ stationId: data.stationId, ...this.queryParams }).then(
        (res) => {
          this.total = res.total;
          this.chargingArr = res.data;
          this.loading = false;
        }
      );
      this.timer = setInterval(() => {
        listPort({ stationId: data.stationId, ...this.queryParams }).then(
          (res) => {
            this.total = res.total;
            this.chargingArr = res.data;
            this.loading = false;
          }
        );
      }, 10000);
    },
    //充电状态转换
    getStatus(status) {
      const typeObject = {
        0: "空闲",
        1: "充电",
        2: "预约",
        3: "正在启动充电",
        10: "启动失败",
        5: "充电故障",
        "05": "充电故障",
      };
      return typeObject[status];
    },

    //充电状态转换
    getPortStatus(status) {
      const typeObject = {
        0: "空闲",
        1: "充电",
        2: "预约",
        3: "正在启动充电",
        10: "启动失败",
        5: "充电故障",
        "05": "充电故障",
      };
      return typeObject[status];
    },
    //模拟桩操作
    operateHandler(key, data) {
      switch (key) {
        case "start":
          this.operateStart(data);
          break;
        case "link":
          this.operateLink(data);
          break;
        case "startCharge":
          this.operateStartCharge(data);
          break;
        case "unlink":
          this.operateUnlink(data);
          break;
        case "stop":
          this.operateStop(data);
          break;
        case "endCharge":
          this.operateEndCharge(data);
          break;

        default:
          break;
      }
    },
    getNewStatus(id) {
      this.loading = true;
      listPort({ stationId: this.stationId, ...this.queryParams }).then(
        (res) => {
          this.total = res.total;
          this.chargingArr = res.data;
          this.loading = false;
        }
      );
    },
    //启动模拟充电桩
    async operateStart(data) {
      try {
        const res = await start({
          pileId: data.pileId,
        });
        if (res.code == 200) {
          this.getNewStatus(data.stationId);
          this.$message.success(res.data);
        } else {
          this.$message.error(res.data);
        }
      } catch (error) {
        console.log("error: ", error);
      }
    },
    //插枪
    async operateLink(data) {
      try {
        const res = await link({
          pileId: data.pileId,
          deviceId: data.deviceId,
        });
        if (res.code == 200) {
          this.getNewStatus(data.stationId);
          this.$message.success(res.data);
        } else {
          this.$message.error(res.data);
        }
      } catch (error) {
        console.log("error: ", error);
      }
    },
    //开始充电
    async operateStartCharge(data) {
      console.log("data: ", data);
      try {
        const res = await startCharge({
          userId: data.userId,
          pileId: data.pileId,
          deviceId: data.deviceId,
        });
        if (res.code == 200) {
          this.getNewStatus(data.stationId);
          this.$message.success(res.data);
        } else {
          this.$message.error(res.data);
        }
      } catch (error) {
        console.log("error: ", error);
      }
    },
    //停止充电
    async operateEndCharge(data) {
      try {
        const res = await endCharge({
          pileId: data.pileId,
          deviceId: data.deviceId,
        });
        if (res.code == 200) {
          this.getNewStatus(data.stationId);
          this.$message.success(res.data);
        } else {
          this.$message.error(res.data);
        }
      } catch (error) {
        console.log("error: ", error);
      }
    },

    //拔枪
    async operateUnlink(data) {
      try {
        const res = await unlink({
          pileId: data.pileId,
          deviceId: data.deviceId,
        });
        if (res.code == 200) {
          this.getNewStatus(data.stationId);
          this.$message.success(res.data);
        } else {
          this.$message.error(res.data);
        }
      } catch (error) {
        console.log("error: ", error);
      }
    },
    //停止模拟器
    async operateStop(data) {
      try {
        const res = await stop({
          pileId: data.pileId,
        });
        if (res.code == 200) {
          this.getNewStatus(data.stationId);
          this.$message.success(res.data);
        } else {
          this.$message.error(res.data);
        }
      } catch (error) {
        console.log("error: ", error);
      }
    },
  },
  created() {},
  mounted() {
    this.getList();
  },
  updated() {},
  destroyed() {},
};
</script>
<style scoped lang="scss">
.content {
  display: flex;
  align-items: center;
  width: 100%;
  height: 100%;
  padding: 10px;
  background-color: #f3f6ff;

  .left {
    width: 250px;
    height: calc(100vh - 105px);
    overflow: auto;
    background-color: #fff;
    border: 1px solid #f1eded;
    border-radius: 2px;
    .left-title {
      height: 50px;
      line-height: 50px;
      font-size: 14px;
      font-weight: 400;
      color: #333;
      padding-left: 10px;
      border-bottom: 1px solid #f4eded;
    }
    .list {
      height: calc(100% - 60px);
      .list-input {
        margin-top: 10px;
      }
      .tree {
        margin-top: 5px;
        height: calc(100% - 50px);
        overflow: auto;
      }
    }
  }
  .right {
    flex: 1;
    height: calc(100vh - 105px);
    background-color: #fff;
    padding: 0 10px;

    .right-title {
      font-size: 22px;
      font-weight: 400;
      height: 50px;
      line-height: 50px;
      color: #333;
    }
    .name {
      display: inline-block;
      background-color: #e8ebf7;
      color: #3156dd;
      padding: 6px 8px;
      border-radius: 5px;
    }
    .list {
      // display: flex;
      // flex-wrap: wrap;
      margin-top: 20px;
      height: calc(100% - 180px);
      overflow: auto;
      .station {
        margin-right: 40px;
        margin-bottom: 30px;
        width: 304px;
        height: 230px;
        display: inline-block;
        background: linear-gradient(8deg, #fafbff, #ffffff);
        border: 1px solid #ebefff;
        box-shadow: 0 0px 5px #d3e0f0;
        position: relative;
        .operateBtn {
          position: absolute;
          bottom: 3px;
          right: 5px;
          font-size: 30rpx;
          cursor: pointer;
          color: #5069e7;
        }
        .info {
          display: flex;
          align-items: center;
          justify-content: space-between;
          width: 100%;
          height: 45px;
          background: #e8ebf7;
          padding: 0 10px;
          .num {
            display: flex;
            align-items: center;
            > img {
              height: 18px;
              margin-right: 5px;
            }
          }
          .status {
            color: #5069e7;
          }
          .status1 {
            color: #eaaf32;
          }
          .status2 {
            color: #ec2c13;
          }
        }
        .mess {
          display: flex;
          flex-wrap: wrap;
          align-items: center;

          .item {
            width: 30%;
            margin-right: 10px;
            margin-top: 30px;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;

            &-value {
              font-size: 18px;
              font-weight: 500;
              color: #333;
              margin-bottom: 5px;
              white-space: nowrap;
            }
            &-name {
              font-size: 14px;
              font-weight: 400;
              color: #666;
            }
          }
        }
      }
    }
  }
}
.operate {
  display: flex;
  flex-direction: column;
  button {
    margin-bottom: 7px;
  }
  ::v-deep .el-button + .el-button {
    margin-left: 0px;
  }
}

::-webkit-scrollbar {
  width: 8px; /* 设置滚动条的宽度 */
  height: 6px;
}

/* 自定义滚动条滑块样式 */
::-webkit-scrollbar-thumb {
  background-color: #b6bfec; /* 设置滑块的颜色 */
  border-radius: 5px; /* 设置滑块的圆角 */
}

/* 自定义滚动条轨道样式 */
::-webkit-scrollbar-track {
  background-color: #f1f1f1; /* 设置轨道的颜色 */
}
</style>
