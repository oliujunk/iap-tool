<template>
  <div class="home">
    <el-row :gutter="20">
      <el-col :span="12">
        <el-form :model="iapSetting" label-position="top">
          <el-form-item label="通信设置">
            <el-form :model="iapSetting.communication" label-width="80px" label-position="top" size="mini">
              <el-form-item label="通信类型">
                <el-radio-group v-model="iapSetting.communication.type" :disabled="iapSetting.communication.open">
                  <el-radio-button :label="0">串口</el-radio-button>
                  <el-radio-button :label="1">TCP Server</el-radio-button>
                  <el-radio-button :label="2">TCP Client</el-radio-button>
                </el-radio-group>
              </el-form-item>

              <el-form-item label="串口号" v-if="iapSetting.communication.type === 0">
                <el-select v-model="iapSetting.communication.serialPort.path" placeholder="请选择">
                  <el-option
                    v-for="item in serialPortList"
                    :key="item.path"
                    :label="item.path"
                    :value="item.path"
                  >
                  </el-option>
                </el-select>
              </el-form-item>

              <el-form-item label="端口" v-if="iapSetting.communication.type === 1">
                <el-input-number v-model="iapSetting.communication.tcpServer.port" :min="1" :max="65535"></el-input-number>
              </el-form-item>

              <el-form-item label="IP" v-if="iapSetting.communication.type === 2">
                <el-input v-model="iapSetting.communication.tcpClient.ip" style="width: 300px;"></el-input>
              </el-form-item>
              <el-form-item label="端口" v-if="iapSetting.communication.type === 2">
                <el-input-number v-model="iapSetting.communication.tcpClient.port" :min="1" :max="65535"></el-input-number>
              </el-form-item>

              <el-form-item>
                <el-button
                  type="danger"
                  @click="onOpenSerialPort"
                  v-if="iapSetting.communication.type === 0 && !iapSetting.communication.open"
                >打开串口</el-button>

                <el-button
                  type="success"
                  @click="onCloseSerialPort"
                  v-if="iapSetting.communication.type === 0 && iapSetting.communication.open"
                >关闭串口</el-button>

                <el-button
                  type="danger"
                  @click="onServerListen"
                  v-if="iapSetting.communication.type === 1 && !iapSetting.communication.open"
                >监听</el-button>

                <el-button
                  type="success"
                  @click="onServerClose"
                  v-if="iapSetting.communication.type === 1 && iapSetting.communication.open"
                >取消监听</el-button>

                <el-button
                  type="danger"
                  @click="onClientConnect"
                  v-if="iapSetting.communication.type === 2 && !iapSetting.communication.open"
                >连接</el-button>

                <el-button
                  type="success"
                  @click="onClientDisconnect"
                  v-if="iapSetting.communication.type === 2 && iapSetting.communication.open"
                >断开连接</el-button>

              </el-form-item>

            </el-form>
          </el-form-item>
          <el-form-item label="文件选择">
            <el-upload
              class="upload-demo"
              ref="upload"
              action=""
              :auto-upload="false"
              accept=".bin"
              :on-change="onSelectFile"
            >
              <el-button slot="trigger" size="small" type="primary">选取文件</el-button>
            </el-upload>
          </el-form-item>
          <el-form-item label="固件更新">
            <el-button type="primary" @click="onDownload">开始下载</el-button>
          </el-form-item>
        </el-form>
      </el-col>
      <el-col :span="12">

      </el-col>
    </el-row>
  </div>
</template>

<script>
import {
  MODEM_EOT,
  MODEM_NAK,
  firstFrame,
  dataFrame,
  lastFrame,
  modbusCrc,
} from '@/common/ymodem';

import { SerialPort } from 'serialport';
// eslint-disable-next-line import/no-extraneous-dependencies
import { InterByteTimeoutParser } from '@serialport/parser-inter-byte-timeout';

// const net = require('net');
const fs = require('fs');

export default {
  name: 'HomeView',

  data() {
    return {
      serialPortList: [],
      iapSetting: {
        communication: {
          type: 0,
          open: false,
          serialPort: {
            path: '',
          },
          tcpServer: {
            port: 8888,
          },
          tcpClient: {
            ip: '',
            port: 8888,
          },
        },
      },
      serialPort: null,
      binaryData: null,
      downloading: false,
      sumPkgNum: 0,
      curPkgNum: -1,
      percentage: 0,
      progressStatus: 'exception',

    };
  },

  methods: {
    listSerialPort() {
      SerialPort.list().then((ports) => {
        this.serialPortList = ports;
      });
    },

    processRecvData(data) {
      console.log('Data:', data);
    },

    onOpenSerialPort() {
      this.serialPort = new SerialPort({
        path: this.iapSetting.communication.serialPort.path,
        baudRate: 9600,
      });

      this.serialPort.on('open', () => {
        this.iapSetting.communication.open = true;
        const parser = this.serialPort.pipe(new InterByteTimeoutParser({ interval: 30 }));
        parser.on('data', this.processRecvData);
      });

      this.serialPort.on('error', (err) => {
        console.log('Error: ', err.message);
        this.$message.error('串口打开失败：', err.message);
      });
    },

    onCloseSerialPort() {
      this.serialPort.close((err) => {
        if (err) {
          this.$message.error('串口关闭失败：', err.message);
        } else {
          this.iapSetting.communication.open = false;
        }
      });
    },

    onServerListen() {},

    onServerClose() {},

    onClientConnect() {},

    onClientDisconnect() {},

    onSelectFile(file) {
      console.log(file);
      const binaryData = fs.readFileSync(file.raw.path);
      this.binaryData = binaryData;
      this.sumPkgNum = Math.ceil(binaryData.length / 128);
    },

    onDownload() {
      this.curPkgNum = -1;
      this.downloading = false;
      this.percentage = 0;
      this.progressStatus = 'exception';
      const send = Buffer.alloc(4);
      send[0] = 0x01;
      send[1] = 0xe0;
      const checkSum = modbusCrc(send, 0, 2);
      send[2] = checkSum & 0xff;
      send[3] = (checkSum >> 8) & 0xff;
      this.textarea += `发送(HEX): ${send.toString('hex').toUpperCase()}\r\n`;
      if (this.linkType === 1) {
        this.port.write(send);
      } else if (this.linkType === 2) {
        this.socket.write(send);
      }
    },

  },

  mounted() {
    this.listSerialPort();
  },
};
</script>

<style lang="scss" scoped>
.home {
  width: 100%;
}

.el-row {
  width: 100%;
  text-align: left;
}

</style>
