---
layout: post
title: 多文件断点上传
tags: 技术
---
<template>
  <div class="upload-wrap" ref="uploadr">
    <el-popover
      placement="top-end"
      title="传输列表"
      width="481"
      trigger="click"
      content="传输列表"
      popper-class="openprorp"
    >
      <div class="bigopen1">
        <div class="topopen_1">
          <p class="topopen_font1">
            正在上传
            <span v-if="openlistnum.length > 0">{{
              "(" + fileNum() + "/" + openlistnum.length + ")"
            }}</span>
          </p>
          <p class="topopen_font2" @click="empty">清空全部</p>
        </div>

        <div class="bottomopen_1">
          <p
            v-if="openlistnum.length == 0"
            class="bottomopen_font1"
            style="margin-top: 20.5px"
          >
            暂无数据
          </p>
          <!-- 上传文件列表 -->
          <div
            v-slse
            v-for="(item, key) in openlistnum"
            :key="`${item.fileMd5}_${item.name}`"
            class="bottomopen_box"
          >
            <!-- 上部分 -->
            <div class="bottomopen_box_center">
              <div class="bottomopen_box_center_left">
                <i class="el-icon-document"></i>
                <!-- <i class="el-icon-folder-opened"></i> -->
                <p class="bottomopen_box_center_filename">
                  {{ item.fileName }}
                </p>
              </div>
              <div class="bottomopen_box_center_right">
                <span v-if="item.status == 1">
                  <span
                    v-if="!item.btn && (item.total != 0 || item.total != 100)"
                  >
                    <el-tooltip
                      class="item"
                      effect="dark"
                      content="暂停上传"
                      placement="bottom"
                    >
                      <span>
                        <i
                          class="el-icon-video-pause"
                          v-if="
                            !item.btn && (item.total != 0 || item.total != 100)
                          "
                          @click="handleBtn(item)"
                        ></i>
                      </span>
                    </el-tooltip>
                  </span>
                  <span
                    v-if="item.btn && (item.total != 0 || item.total != 100)"
                  >
                    <el-tooltip
                      class="item"
                      effect="dark"
                      content="继续上传"
                      placement="bottom"
                    >
                      <span>
                        <i
                          class="el-icon-video-play"
                          v-if="
                            item.btn && (item.total != 0 || item.total != 100)
                          "
                          @click="handleBtn(item)"
                        ></i>
                      </span>
                    </el-tooltip>
                  </span>
                </span>
                <span v-else>
                  <!-- 上传成功 -->
                  <i class="el-icon-circle-check" v-if="item.total == 100"></i>
                  <!-- 上传失败 -->
                  <i
                    class="el-icon-warning-outline"
                    style="color: rgb(255, 153, 0)"
                    v-if="item.uploadErr"
                  ></i>
                  <el-tooltip
                    class="item"
                    effect="dark"
                    content="清空"
                    placement="bottom"
                  >
                    <span>
                      <i
                        class="el-icon-delete icon_delete_1"
                        @click="handleDelete(key)"
                      ></i>
                    </span>
                  </el-tooltip>
                </span>
              </div>
            </div>
            <!-- 中间步骤条 -->
            <div class="bottomopen_box_center1">
              <el-progress
                :show-text="false"
                :stroke-width="3"
                :percentage="item.total"
                class="bottomopen_box_center_progress"
              />
            </div>
            <!-- 下部分 -->
            <div class="bottomopen_box_center2">
              <p>{{ filesizeConvert(item.size) }}</p>
              <p>{{ internetSpeed }}</p>
            </div>
          </div>
          <!-- 上传文件列表结束 -->
        </div>
      </div>
      <el-button type="primary" size="small" slot="reference"
        >传输列表</el-button
      >
    </el-popover>
    <el-upload
      class="upload-demo"
      ref="adModel"
      action="#"
      :headers="headers"
      :on-success="handleUploadSuccess"
      :on-error="handleerror"
      :file-list="fileList"
      :on-change="changeUpload"
      :before-upload="beforeUpload"
      :http-request="uploadHttpRequest"
    >
      <el-button size="small" type="primary" @click="upload">上传</el-button>
    </el-upload>
  </div>
</template>
<script>
import SparkMD5 from "spark-md5";
import {
  upLoadFile,
  breakpointUpLoadFile,
  checkFileMd5,
  breakpointGetKey,
} from "@/api/filestorage/index";
import { fileParse } from "@/utils/parseFile";
import { getCookie } from "@/utils/cookies";
import { paramSeparator } from "../el-data-table/utils/query";
export default {
  name: "PublicNetworkIp",
  props: ["upid", "parentid", "filePath", "goHistory", "disabled"],
  data() {
    return {
      uploadUrl: "",
      uploadKey: "",
      mewfilePath: "",
      fileList: [],
      USER_id: sessionStorage.getItem("USER_ID"),
      file: "",
      action: "",
      loading: false,
      headers: {
        token: getCookie("token"),
        "Content-Type": "application/json;charset=UTF-8",
      },
      uploadDisabled: false,
      timer: null,
      // 分片上传
      total: 0,
      btn: false,
      abort: false,
      uploadSuc: false,
      slicesNum: null,
      chunkNumber: null,
      checkUrl: "",
      netTimer: null,
      internetSpeed: 0,
      isUploading: false, // 开始上传标示
      openlistnum: [],
      randomName: "", //随机文件名
    };
  },
  filters: {
    btnText(btn) {
      return btn ? "继续" : "暂停";
    },
    totalText(total) {
      return total > 100 ? 100 : total;
    },
  },

  watch: {
    upid: {
      handler(newvalue) {
        if (!this.goHistory) {
          this.mewfilePath = this.filePath;
        }
      },
      immediate: true,
      deep: true,
    },
    goHistory: {
      handler(newvalue) {
        if (!newvalue) {
          this.mewfilePath = this.filePath;
        }
      },
    },
    filePath(a) {
      this.mewfilePath = a;
    },
  },
  mounted() {
    this.mewfilePath = this.filePath;
    // 定时获取当前网速
    if (navigator.onLine) {
      this.netTimer = setInterval(() => {
        this.internetSpeed = navigator.connection.downlink + "MB/s";
      }, 1000);
    }
  },
  beforeDestroy() {
    clearInterval(this.netTimer);
    this.netTimer = null;
  },
  methods: {
    upload() {
      this.$emit("setUploadInfo", this.$data);
    },
    // 5秒后关闭
    closeUploadInfo() {
      this.timer = setTimeout(() => {
        this.isUploading = false;
      }, 5000);
    },

    // 分片上传
    async changeFile(file,uploadUrl, reqKeyData) {
      if (!file) return;
      file = file.raw;
      //   this.total = 0;
      //   this.abort = false;
      //   this.btn = false;
      let buffer = await fileParse(file, "buffer"),
        spark = new SparkMD5.ArrayBuffer(),
        hash,
        suffix;
      spark.append(buffer);
      hash = spark.end();
      suffix = /\.([0-9a-zA-Z]+)$/i.exec(file.name)[1];
      
      let multiple = null;
      /**
       * 方案一
       */
      if (file.size <= 5 * 1024 * 1024) {
        //文件不大于5mb 设置切片为一个
        multiple = 1;
      } else {
        for (let i = 20; i >= 1; i--) {
          if (file.size % i == 0) {
            multiple = i;
            break;
          }
        }
      }
      /**
       * 方案二
       */
      const chunkSize = 5 * 1024 * 1024; // 每个块的大小为 5MB
      // const chunkSize = 10; // 每个块的大小为 10
      const fileSize = file.size; // 文件大小
      const chunks = Math.ceil(fileSize / chunkSize); // 总块数

      // 切片个数
      this.slicesNum = chunks || multiple;

      // 创建切片
      let partList = [],
        partsize = chunkSize || file.size / this.slicesNum;
      // cur = 0;
      for (let i = 1; i <= this.slicesNum; i++) {
        const start = (i - 1) * partsize;
        const end = Math.min(start + partsize, fileSize);
        let item = {
          // chunk: file.slice(cur, Math.min(file.size, cur + partsize)),
          chunk: file.slice(start, end),
          filename: `${hash}_${i}.${suffix}`,
          chunkNumber: `${i - 1}`,
          // file: file.slice(cur, Math.min(file.size, cur + partsize)),
          file: file.slice(start, end),
        };
        // cur += partsize;
        partList.push(item);
      }
      this.requestData = {
        fileMd5: hash,
        name: this.randomName,
        size: file.size,
        totalChunks: this.slicesNum,
        chunkSize: partsize,
        fileName: file.name,
        uploadKey: reqKeyData.key, // 后台需要的key
      };
      let obj = {
        total: 0,
        abort: false,
        btn: false,
        partList: partList,
        uploadErr: false, //是否上传失败
        status: 1, //上传状态
      };
      let fileObj = { ...this.requestData, ...obj };
      //文件列表
      this.openlistnum.push(fileObj);

      //   this.partList = partList;
      //   const checkRes = await checkFileMd5(this.checkUrl, {
      //     key: this.uploadKey,
      //     fileName: this.requestData.name,
      //     md5: this.requestData.fileMd5,
      //   }).catch((err) => {
      //     console.log("err--", err);
      //   });
      //   const { data, code } = checkRes;

      //   if (code != 200 && code != 206 && code != 207) {
      //     this.uploadDisabled = false;
      //     return;
      //   }
      //   if (code == 200) {
      //     this.uploadSuc = true;
      //     this.uploadDisabled = false;
      //     this.total = 100;
      //     this.$message.success("此文件已经上传，无需再次上传！");
      //     return;
      //   }

      //   const nextChunkNumber = code == 206 ? data.chunkNumber : 0;

      //   if (nextChunkNumber) {
      //     this.partList.splice(nextChunkNumber - 1, 1);
      //   }

      this.isUploading = true;
      this.sendRequest(fileObj);
    },
    async sendRequest(element) {
      element.uploadSuc = false;
      // 根据100个切片创造100个请求集合
      let requestList = [];
      try {
        element.partList.forEach((item, index) => {
          // 每一个函数都发送一个切片请求
          let fn = async (chunkNumber) => {
            let formData = new FormData(),
              shardFile = new SparkMD5.ArrayBuffer(),
              shardFileBuffer = await fileParse(item.chunk, "buffer"),
              shardFileHash;
            shardFile.append(shardFileBuffer);
            shardFileHash = shardFile.end();
            formData.append(
              "chunk",
               element.reqNextChunkNumber ?  element.reqNextChunkNumber : item.chunkNumber
            );

            formData.append("file", item.file);
            formData.append("chunks", element.totalChunks);
            formData.append("md5", element.fileMd5);
            formData.append("name", element.name);
            formData.append("size", element.chunkSize);
            formData.append("key", element.uploadKey);
            return breakpointUpLoadFile(this.uploadUrl, formData).then(
              (res) => {
                const { code, data } = res;
                if (code == 206) {
                  // element.total += parseInt(100 / element.totalChunks);
                  //进度
                  element.total = parseInt(
                    (res.data.chunkNumber / element.totalChunks) * 100
                  );
                  this.$forceUpdate(); // 强制重新渲染组件
                  // 传完的切片我们把它移除掉
                  if(data.chunkNumber){
                    element.partList.shift();    
                    element.reqNextChunkNumber = res.data.chunkNumber          
                  }
                  return data.chunkNumber;
                } else if (code == 200) {
                  this.$message.success("上传成功");
                  element.uploadSuc = true;
                  element.total = 100;
                  element.status = 0;
                  element.uploadErr = false;
                  this.$parent.gettableList();
                  this.$parent.getPage(2, this.upid, this.USER_id);
                  this.closeUploadInfo();
                } else {
                  throw Error(`We've found the target element.`);
                }
              }
            );
          };
          requestList.push(fn);
        });
      } catch (err) {}

      let i = 0;

      let send = async (params) => {
        if (params.abort) return;
        if (i >= requestList.length && !params.uploadSuc) {
          this.$message.warning("上传失败,请重新上传");
          params.total = 0;
          params.status = 0;
          params.uploadErr = true;
          this.closeUploadInfo();
          return;
        }
        if (params.uploadSuc) {
          params.status = 0;
          params.total = 100;
          params.uploadErr = false;

          return;
        }

        await requestList[i]().then((res) => {
          
        });

        i++;
        send(params);
      };
      send(element);
    },
    handleBtn(params) {
      if (params.btn) {
        params.abort = false;
        params.btn = false;
        this.sendRequest(params);
        return;
      }
      params.btn = true;
      params.abort = true;
    },

    handleUpload(file) {
      this.file = file.file;
    },

    beforeUpload(file) {
      const isLt2M = file.size / 1024 / 1024 < 1024;
      if (!isLt2M) {
        this.$message.error(
          "上传文件大小不能超过 1GB! 如有扩容需要，请联系客服。"
        );
      }
      return isLt2M;
    },
    // 只能上传单个文件，选择文件覆盖原有文件
    changeUpload(file, fileList) {
      if (fileList.length > 0) {
        this.fileList = [fileList[fileList.length - 1]];
      }
    },

    uploadHttpRequest(data) {
      var obj = {
        filePath: this.mewfilePath ? this.mewfilePath : "",
        userId: this.USER_id,
        regionId: this.upid,
        fileName: this.fileList[0].name,
      };
      this.getKeyHanlder(obj, this.fileList[0], data);
    },

    handleUploadSuccess(res, file) {
      this.file = file.raw;
      this.$parent.gettableList();
      this.$parent.getPage(2, this.upid, this.USER_id);
      this.$message.success("上传成功!");
    },
    handleerror(res, file) {
      this.$message.error("上传失败");
    },
    clearTimeoutHandle() {
      clearTimeout(this.timer);
      this.timer = null;
    },

    // 获取key方法
    getKeyHanlder(obj, file, val) {
      //   this.uploadDisabled = true;
      if (this.timer) {
        this.clearTimeoutHandle();
      }
      breakpointGetKey(obj).then((res) => {
        if (res.code == "200") {
          this.uploadUrl = res.data.uploadUrl;
          this.checkUrl = res.data.checkUrl;
          this.uploadKey = res.data.key;
          this.randomName = res.data.fileName;
          var formData = new FormData(); // 当前为空
          formData.append("file", file.raw);
          formData.append("key", this.uploadKey);
          formData.append("regionId", this.upid);
          this.changeFile(file, this.uploadUrl,res.data);
        } else {
          //   this.uploadDisabled = false;
        }
      });
    },
    //删除文件列表
    handleDelete(key) {
      this.openlistnum.splice(key, 1);
    },
    //文件单位转换
    filesizeConvert(size) {
      var size = parseInt(size);
      var data = "";
      if (size < 0.1 * 1024) {
        data = size.toFixed(2) + "B";
      } else if (size < 0.1 * 1024 * 1024) {
        data = (size / 1024).toFixed(2) + "KB";
      } else if (size < 0.1 * 1024 * 1024 * 1024) {
        data = (size / (1024 * 1024)).toFixed(2) + "MB";
      } else {
        data = (size / (1024 * 1024 * 1024)).toFixed(2) + "GB";
      }
      var sizestr = data + "";
      var len = sizestr.indexOf(".");
      var dec = sizestr.substr(len + 1, 2);
      if (dec == "00") {
        return sizestr.substring(0, len) + sizestr.substr(len + 3, 2);
      }
      return sizestr;
    },
    //清空全部
    empty() {
      this.openlistnum = [];
    },
    //正在上传的文件数量
    fileNum() {
      let n = 0;
      this.openlistnum.forEach((e) => {
        if (e.status == 1) {
          n += 1;
        }
      });
      return n;
    },
  },
};
</script>
<style scoped lang="less">
.upload-wrap {
  .progress-wrap {
    margin-right: 20px;
  }
  .upload-demo {
    display: inline-block;
    margin-left: 10px;
  }
}
/*传输列表弹窗*/
.openprorp {
  padding: 12px 16px;
}
.openprorp .el-popover__title {
  font-size: 14px !important;
}
.bigopen1 {
  width: 100%;
  font-size: 14px;
  display: flex;
  flex-direction: column;
}
.topopen_1 {
  width: 451px;
  height: 36px;
  padding: 0px 16px;
  margin-bottom: 12px;
  background-color: #f5f7fa;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.topopen_font1 {
  color: #303133;
}
.topopen_font2 {
  color: #4686f2;
  cursor: pointer;
}
.bottomopen_1 {
  width: 451px;
  min-height: 71px;
  max-height: 270px;
  /* line-height: 71px; */
  padding: 0px 16px;
  display: flex;
  flex-direction: column;
  align-items: center;
  overflow-x: hidden;
  overflow-y: auto;
}
.bottomopen_font1 {
  color: #909399;
}
.bottomopen_box {
  width: 100%;
  height: 71px;
  padding-left: 18px;
  padding-right: 13px;
  border-bottom: 1px solid #f2f5fc;
  display: flex;
  flex-direction: column;
}
/* 传输列表上部分样式 */
.bottomopen_box_center {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 6px;
  margin-bottom: 6px;
}
.bottomopen_box_center_left {
  display: flex;
  align-items: center;
}
.bottomopen_box_center_right {
  font-size: 16px;
  color: #4686f2;
  width: 48px;
  height: 16px;
  display: flex;
  justify-content: flex-end;
  i {
    cursor: pointer;
  }
}
.icon_delete_1 {
  margin-left: 16px;
}
.bottomopen_box_center_filename {
  margin-left: 8px;
}
.bottomopen_box_center1,
.bottomopen_box_center2 {
  margin-left: 22px;
}
/* 传输列表中间样式 */
.bottomopen_box_center1 {
  width: 361px;
}
::v-deep .bottomopen_box_center_progress {
  width: 100% !important;
}
/* 传输列表下部分样式 */
.bottomopen_box_center2 {
  width: 329px;
  font-size: 12px;
  color: #909399;
  display: flex;
  justify-content: space-between;
  margin-top: 8px;
  margin-bottom: 6px;
}
</style>
