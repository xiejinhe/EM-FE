<template>
  <div class="emfe-upload" :class="uploadName">
    <template v-if="type === 'icon'">
      <emfe-button :disabled="disabled || !canUpload" v-show="!src" :theme="theme" type="shangchuan" :className="iconName">{{
          iconText }}</emfe-button>
      <input v-show="!src" class="emfe-upload-file" :class="fileName" :disabled="disabled || !canUpload" type="file" :accept="fileType" @change="change" ref="upload">
      <div v-show="src" :style="{opacity: canShow ? 1 : 0}" class="emfe-upload-icon-wrap">
        <div class="emfe-upload-icon-wrap-box" :class="[`emfe-upload-icon-wrap-box-${align}`, imageName]">
          <img class="emfe-upload-icon-wrap-box-img" :class="[`emfe-upload-img-${align}`, imgName]" :src="src" ref="img">
        </div>
        <i class="emfe-upload-icon-wrap-close" @click="closeFn"></i>
      </div>
    </template>
    <template v-if="type === 'plus'">
      <span v-show="!src" class="emfe-upload-btn" :class="btnName">{{ plusText }}</span>
      <input v-show="!src" class="emfe-upload-file" :class="fileName" :disabled="disabled || !canUpload" type="file" :accept="fileType" @change="change" ref="uploadPlus">
      <div v-show="src" class="emfe-upload-plus-box" :class="[`emfe-upload-plus-box-${align}`, imageName, {'emfe-upload-plus-box-disabled': disabled}]" :style="{opacity: canShow ? 1 : 0}">
        <img :class="[`emfe-upload-img-${align}`, imgName]" v-show="src" :src="src" ref="img">
        <i :class="['emfe-upload-plus-close', {'emfe-upload-plus-box-disabled': disabled}]" @click="closePlusFn"></i>
      </div>
    </template>
    <emfe-modal :show="interceptModal" title="截取器" @close="formCancel" @cancel="formCancel" @ok="formOk" okText="保存" className="form">
      <div slot="modal-main" class="emfe-upload-intercept-wrap" :style="{'padding-top': `${dragPaddingTop}px`, 'padding-left': `${dragPaddingLeft}px`}">
        <emfe-drag class="emfe-upload-intercept-drag" :dragEl="drag1" :style="{ width: `${dragWidth}px`, height: `${dragHeight}px`}" :initialValue="[-interceptCanvasWidth/2, -interceptCanvasHeight/2]" limit="true" @drag="dragPosMove" :padding="[dragPaddingLeft, dragPaddingTop]">
          <img :src="img" class="emfe-upload-intercept-img" :style="{ width: `${dragWidth}px`, height: `${dragHeight}px`}" ref="previewImg">
          <div class="emfe-upload-intercept" :style="{width: `${interceptCanvasWidth}px`, height: `${interceptCanvasHeight}px`, left: `${interceptLeft}px`, top: `${interceptTop}px`}" ref="drag1">
            <!-- 左上 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-nw" @drag="nwPosMove" @afterDrag="upDot" :moveEle="false"></emfe-drag>
            <!-- 右上 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-ne" @drag="nePosMove" @afterDrag="upDot" :moveEle="false"></emfe-drag>
            <!-- 左下 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-sw" @drag="swPosMove" @afterDrag="upDot" :moveEle="false"></emfe-drag>
            <!-- 右下 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-se" @drag="sePosMove" @afterDrag="upDot" :moveEle="false"></emfe-drag>
            <!-- 右下 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-se" @drag="sePosMove" @afterDrag="upDot" :moveEle="false"></emfe-drag>
            <!-- 上 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-top" @drag="topPosMove" @afterDrag="upDot" :moveEle="false" v-if="!interceptSync"></emfe-drag>
            <!-- 右 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-right" @drag="rightPosMove" @afterDrag="upDot" :moveEle="false" v-if="!interceptSync"></emfe-drag>
            <!-- 下 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-bottom" @drag="bottomPosMove" @afterDrag="upDot" :moveEle="false" v-if="!interceptSync"></emfe-drag>
            <!-- 左 -->
            <emfe-drag class="emfe-upload-intercept-point emfe-upload-intercept-point-left" @drag="leftPosMove" @afterDrag="upDot" :moveEle="false" v-if="!interceptSync"></emfe-drag>
          </div>
        </emfe-drag>
      </div>
    </emfe-modal>
  </div>
</template>
<script>
import _ from '../../../tools/lodash';
import upload from '../../../tools/upload';
import { getElementLeft, getElementTop } from '../../../tools/assist';
import { openMask, closeMask } from '../../../tools/body';
import EmfeMessage from '../../emfe-message/index';
import ajax from './ajax';
import CONSTANT from '../../../contant';

let pointOldLeft = 0; // 改变截取器遮罩大小
let nenwTopOld = 0; // 改变截取器遮罩大小， 只用于 右上角在不等比的时候
const iconBoxHeight = 70; //icon模式外框的高度
const iconBoxWidth = 118; //icon模式外框的宽度
let firstDot = false; // 是否是第一次用点改变大小

export default {
  name: 'upload',
  data() {
    return {
      uploadJpeg: `image/${this.interceptType}`,
      drag1: [],
      canUpload: true,
      src: '',
      canShow: false,
      fileList: [],
      tempIndex: 1,
      img: '',
      align: '',
      interceptModal: false, // 截取器是否显示
      interceptWidth: 360, // 截取器的宽
      interceptHeight: 400, // 截取器的搞
      interceptLeft: 0, // 截取器的左边距离
      interceptTop: 0, // 截取器的有边距离
      interceptCanvasWidth: 360, // 截取器截图的大小
      interceptCanvasHeight: 400, // 截取器截图的大小
      dragWidth: 'auto', // 拖拽的宽
      dragHeight: 400, // 拖拽的高
      dragPaddingLeft: 0,
      dragPaddingTop: 0,
      canvas: null,
      canvasContext: null,
      iconText: this.buttonText,
      plusText: '+',
    };
  },
  props: {
    type: {
      validator(value) {
        return _.has(value, ['plus', 'icon']);
      },
      default: 'plus',
    },
    disabled: Boolean,
    className: {
      type: String,
      default: '',
    },
    action: {
      type: String,
      required: true,
    },
    headers: {
      type: Object,
      default() {
        return {};
      },
    },
    data: {
      type: Object,
    },
    name: {
      type: String,
      default: 'file',
    },
    withCredentials: {
      type: Boolean,
      default: false,
    },
    intercept: {
      type: Array,
      default: () => [],
    },
    interceptType: {
      type: String,
      default: 'jpeg',
    },
    interceptSync: { // 是否等比缩放
      type: Boolean,
      default: true,
    },
    format: {
      type: Array,
      default() {
        return [];
      },
    },
    maxSize: {
      type: Number,
    },
    url: String,
    success: {
      type: Function,
      default: () => {},
    },
    exceededSize: {
      type: Function,
      default: () => {},
    },
    formatError: {
      type: Function,
      default: () => {},
    },
    beforeUpload: {
      type: Function,
      default: () => {},
    },
    error: {
      type: Function,
      default: () => {},
    },
    close: {
      type: Function,
      default: () => {},
    },
    buttonText: {
      type: String,
      default: '上传图片',
    },
    theme: {
      type: String,
      default: 'default',
    },
    fileType: {
      type: String,
      default: 'image/*',
    },
  },
  computed: {
    uploadName() {
      return [
        `emfe-upload-${this.type}`,
        {
          [`${this.className}-upload`]: !!this.className,
        },
        {
          [`${this.className}-upload-${this.type}`]: !!this.className,
        },
        {
          'emfe-upload-disabled': this.disabled || !this.canUpload,
        },
        {
          [`${this.className}-upload-disabled`]: (this.disabled || !this.canUpload) && !!this.className,
        },
      ];
    },
    iconName() {
      return this.className ? `${this.className}-upload-icon-btn` : '';
    },
    btnName() {
      return [
        [`emfe-upload-${this.type}-btn`],
        {
          [`${this.className}-upload-${this.type}-btn`]: !!this.className,
        },
      ];
    },
    imageName() {
      return [
        [`emfe-upload-${this.type}-image`],
        {
          [`${this.className}-upload-${this.type}-image`]: !!this.className,
        },
      ];
    },
    imgName() {
      return [
        [`emfe-upload-${this.type}-image`],
        {
          [`${this.className}-upload-${this.type}-img`]: !!this.className,
        },
      ];
    },
    fileName() {
      return [
        [`emfe-upload-${this.type}-file`],
        {
          [`${this.className}-upload-${this.type}-file`]: !!this.className,
        },
        {
          'emfe-upload-file-disabled': this.disabled || !this.canUpload,
        },
      ];
    },
  },
  mounted() {
    if (this.url) {
      this.initImg(this.url);
    }
    this.initIntercept();
  },
  methods: {
    initIntercept() {
      // 有截取器
      if (this.intercept.length > 0) {
        if (this.intercept.length > 1) {
          this.interceptCanvasWidth = this.intercept[0];
          this.interceptCanvasHeight = this.intercept[1];
        } else if (this.intercept.length === 1) {
          this.interceptCanvasWidth = this.intercept[0];
          this.interceptCanvasHeight = this.intercept[0];
        }
      }
    },
    initImg(val) {
      if (val) {
        const imgObject = new Image();
        imgObject.src = val;
        imgObject.onload = () => {
          this.src = this.url;
          setTimeout(this.setAlign.bind(this), 0);
        };
      } else {
        this.src = '';
      }
    },
    openInterceptModal() {
      openMask();
      this.interceptModal = true;
    },
    closeInterceptModal() {
      closeMask();
      this.interceptModal = false;
    },
    formCancel() {
      this.closeInterceptModal();
      this.resetInputFile();
    },
    formOk() {
      this.initCanvas();
    },
    initCanvas() {
      const img = new Image();
      img.src = this.img;
      img.onload = () => {
        this.canvas = document.createElement('canvas');
        this.canvasContext = this.canvas.getContext('2d');
        this.canvas.width = this.interceptCanvasWidth;
        this.canvas.height = this.interceptCanvasHeight;
        this.getImageUrl();
        this.postHandle(upload(this.clipData, this.uploadJpeg));
        this.closeInterceptModal();

        // const { clientWidth, clienHegiht } = this.$refs.previewImg;
        // this.interceptLeft = clientWidth / 2;
        // this.interceptTop = clienHegiht / 2;
      };
    },
    getImageUrl() {
      const { canvasContext, previewImg } = this;
      const { clientWidth, clientHeight } = previewImg;
      const left = this.interceptLeft;
      const top = this.interceptTop;
      canvasContext.drawImage(previewImg, -left, -top, clientWidth, clientHeight);
      this.clipData = this.canvas.toDataURL(this.uploadJpeg);
    },
    // 拖拽大方块改变截图位置
    dragPosMove(ev, left, top) {
      this.interceptLeft = left;
      this.interceptTop = top;
      if (this.interceptLeft < 5) {
        this.interceptLeft = 2.5;
      }
    },
    // 角部拖拽改变大小
    pointMoveChangeSize(ev, left, mouseLeft, top, mouseTop, lDir, type) {
      const cWidth = this.interceptCanvasWidth;
      const cHeight = this.interceptCanvasHeight;
      let widthStep = -left;
      let heightStep = -top;
      // 下
      if (type === 'top') { // 左上
        const lChangeTop = 5 + mouseTop;
        heightStep = nenwTopOld - lChangeTop;
        nenwTopOld = lChangeTop;
        // 第二次改变的时候，修复恢复原位
        if (firstDot) {
          firstDot = false;
          heightStep = lChangeTop > 0 ? 1 : -1;
        }
      } else if (type === 'right') { // 如果是右
        const lChange = left - (cWidth - 5);
        widthStep = lChange;
        // 第二次改变的时候，修复恢复原位
        if (firstDot) {
          firstDot = false;
          widthStep = lChange > 0 ? 1 : -1;
        }
      } else if (type === 'bottom') {
        const lChangeTop = top - (cHeight - 5);
        heightStep = lChangeTop;
        // 第二次改变的时候，修复恢复原位
        if (firstDot) {
          firstDot = false;
          heightStep = lChangeTop > 0 ? 1 : -1;
        }
      } else if (type === 'left') { // 左
        const lChange = pointOldLeft - left;
        widthStep = lChange;
        // 第二次改变的时候，修复恢复原位
        if (firstDot) {
          firstDot = false;
          widthStep = lChange > 0 ? 1 : -1;
        }
      } else if (type === 'sw') { // 左下
        const lChange = pointOldLeft - left;
        const lChangeTop = top - (cHeight - 5);
        widthStep = lChange;
        heightStep = lChangeTop;
        // 第二次改变的时候，修复恢复原位
        if (firstDot) {
          firstDot = false;
          widthStep = lChange > 0 ? 1 : -1;
          heightStep = lChangeTop > 0 ? 1 : -1;
        }
      } else if (type === 'nw') { // 左上
        const lChange = pointOldLeft - left;
        widthStep = lChange;
        const lChangeTop = 5 + mouseTop;
        heightStep = nenwTopOld - lChangeTop;
        nenwTopOld = lChangeTop;
        // 第二次改变的时候，修复恢复原位
        if (firstDot) {
          firstDot = false;
          widthStep = lChange > 0 ? 1 : -1;
          heightStep = lChangeTop > 0 ? 1 : -1;
        }
      } else if (type === 'ne') { // 如果是右上
        const lChangeTop = 5 + mouseTop;
        heightStep = nenwTopOld - lChangeTop;
        nenwTopOld = lChangeTop;
        // 第二次改变的时候，修复恢复原位
        if (firstDot) {
          firstDot = false;
          heightStep = lChangeTop > 0 ? 1 : -1;
        }
      }
      // 如果等比， 高度直接等比缩放
      if (this.interceptSync) {
        heightStep = (cHeight * widthStep) / cWidth;
      }

      const canWidth = this.intercept[0] < cWidth + widthStep;
      const canHeight = this.intercept[1] < cHeight + heightStep;
      // 拖动左边的，x不能超出去，拖动右边的，宽度不能超过去
      const heng = mouseLeft + this.interceptLeft <= this.dragWidth - 5;
      const shu = mouseTop + this.interceptTop <= this.dragHeight - 5;

      if (this.interceptSync) { // 如果等比
        if (heng && shu && canWidth && canHeight &&
          getElementLeft(this.$refs.previewImg) < ev.clientX - 3 &&
          getElementTop(this.$refs.previewImg) < ev.clientY - 3) {
          this.interceptCanvasWidth += widthStep;
          this.interceptCanvasHeight += heightStep;
        }
      } else {
        if (heng && canWidth && (this.interceptLeft > 2.5 || type === 'se' || type === 'right')) {
          this.interceptCanvasWidth += widthStep;
        }

        if (shu && canHeight && (this.interceptLeft > 2.5 || type === 'se') && this.interceptTop >= 0) {
          this.interceptCanvasHeight += heightStep;
        }
      }

      // 改变左边位置
      // 左上 || 左下
      if (this.interceptSync) {
        if (heng && shu &&
          getElementLeft(this.$refs.previewImg) < ev.clientX - 3 &&
          getElementTop(this.$refs.previewImg) < ev.clientY - 3 &&
          canWidth && canHeight) {
          // 左上 || 左下
          if ((type === 'nw' || type === 'sw')) {
            this.interceptLeft -= widthStep;
            if (this.interceptLeft < 5) {
              this.interceptLeft = 2.5;
            }
          }
          // 左上 || 右上
          if ((type === 'nw' || type === 'ne')) {
            this.interceptTop -= heightStep;
            if (this.interceptTop < 5) {
              this.interceptTop = 2.5;
            }
          }
          pointOldLeft = left;
        }
      } else {
        if (canWidth &&
          (type === 'nw' || type === 'sw' || type === 'left')) {
          this.interceptLeft -= widthStep;
          if (this.interceptLeft < 5) {
            this.interceptLeft = 2.5;
          }
          pointOldLeft = left;
        }
        // 左上 || 右上
        if (
          canHeight &&
          (type === 'nw' || type === 'ne' || type === 'top') &&
          (this.interceptLeft > 2.5 || type === 'se') &&
          this.interceptTop > 2.5) {
          this.interceptTop -= heightStep;
        }
      }
    },
    // 点抬起
    upDot() {
      firstDot = true;
    },
    // 左上
    nwPosMove(ev, left, top, lDir) {
      this.pointMoveChangeSize(ev, left, left, top, top, lDir, 'nw');
    },
    // 右上
    nePosMove(ev, left, top, lDir) {
      this.pointMoveChangeSize(ev, this.interceptCanvasWidth - left, left, this.interceptCanvasHeight - top, top, lDir, 'ne');
    },
    // 左下
    swPosMove(ev, left, top, lDir) {
      this.pointMoveChangeSize(ev, left, left, top, top, lDir, 'sw');
    },
    // 右下
    sePosMove(ev, left, top, lDir) {
      this.pointMoveChangeSize(ev, this.interceptCanvasWidth - left, left, this.interceptCanvasHeight - top, top, lDir, 'se');
    },
    // 上
    topPosMove(ev, left, top, lDir) {
      this.pointMoveChangeSize(ev, left, left, top, top, lDir, 'top');
    },
    // 右
    rightPosMove(ev, left, top, lDir) {
      this.pointMoveChangeSize(ev, left, left, top, top, lDir, 'right');
    },
    // 下
    bottomPosMove(ev, left, top, lDir) {
      this.pointMoveChangeSize(ev, left, left, top, top, lDir, 'bottom');
    },
    // 左
    leftPosMove(ev, left, top, lDir) {
      this.pointMoveChangeSize(ev, left, left, top, top, lDir, 'left');
    },
    setAlign(res) {
      const { clientWidth, clientHeight } = this.$refs.img;
      const width = res ? res.width : clientWidth;
      const height = res ? res.height : clientHeight;
      if (width !== 0 && height !== 0) {
        if (width > height) {
          const newImgScale = iconBoxWidth / width;
          // 如果按宽度适配，图片的高度超出线框
          if (height * newImgScale > iconBoxHeight) {
            this.align = 'vertical';
          } else {
            this.align = 'horizontal';
          }
        } else if (width < height) {
          this.align = 'vertical';
        } else {
          this.align = 'normal';
        }
      }
      this.canShow = true;
    },
    change(e) {
      const files = e.target.files;
      this.$emit('getfileName', e.target.files);

      if (!files) {
        return;
      }

      const postFiles = Array.prototype.slice.call(files);

      if (this.intercept.length === 0) {
        postFiles.forEach((file) => {
          this.postHandle(file);
        });
      } else {
        const reader = new FileReader();
        reader.readAsDataURL(postFiles[0]);
        reader.onload = (readerEvent) => {
          const myImage = new Image();
          myImage.src = readerEvent.target.result;

          myImage.onload = () => {
            this.openInterceptModal();
            this.img = readerEvent.target.result;
            setTimeout(() => {
              this.drag1.length = 0;
              this.drag1.push(this.$refs.drag1);
              this.previewImg = this.$refs.previewImg;
              this.dragWidth = this.previewImg.clientWidth;
              this.dragHeight = this.previewImg.clientHeight;
              this.dragPaddingLeft = (this.dragWidth - this.interceptWidth) / 2;
              this.dragPaddingTop = (this.dragHeight - this.interceptHeight) / 2;
              const imgScale = this.dragHeight / this.dragWidth;
              // 如果宽度超了
              if (this.dragWidth > this.interceptWidth) {
                this.dragWidth = this.interceptWidth;
                this.dragHeight = imgScale * this.dragWidth;
                this.dragPaddingLeft = (this.interceptWidth - this.dragWidth) / 2;
                this.dragPaddingTop = (this.interceptHeight - this.dragHeight) / 2;
              }
              // 如果框框超出去了
              const interceptCanvasScale = this.interceptCanvasWidth / this.interceptCanvasHeight;
              if (this.dragWidth < this.interceptCanvasWidth) {
                this.interceptCanvasWidth = this.dragWidth;
                this.interceptCanvasHeight = interceptCanvasScale / this.interceptCanvasWidth;
              }

              if (this.dragHeight < this.interceptCanvasHeight) {
                this.interceptCanvasHeight = this.dragHeight;
                this.interceptCanvasWidth = interceptCanvasScale / this.interceptCanvasHeight;
              }
              // 展示图片，截图的框框初始化在中间
              const dragWidthNew = this.dragWidth - this.intercept[0];
              const dragHeightNew = this.dragHeight - this.intercept[1];
              this.interceptLeft = dragWidthNew > 0 ? dragWidthNew / 2 : 0;
              this.interceptTop = dragHeightNew > 0 ? dragHeightNew / 2 : 0;
              // 如果图片小于截取器的尺寸, 针对 高度小于截取器的情况
              if (this.interceptCanvasHeight < this.intercept[1]) {
                this.interceptCanvasWidth = this.interceptCanvasHeight * interceptCanvasScale;
              }
            }, 0);
          };
        };
      }
    },
    postHandle(file) {
      // check format
      if (this.format.length) {
        const fileFormat = file.name.split('.').pop().toLocaleLowerCase();
        const checked = this.format.some(item => item.toLocaleLowerCase() === fileFormat);
        if (!checked) {
          this.formatError(file, this.fileList, EmfeMessage);
          this.$emit('formatError', file, this.fileList, EmfeMessage);
          return false;
        }
      }
      // check maxSize
      if (this.maxSize) {
        if (file.size > this.maxSize * 1024) {
          this.exceededSize(file, this.fileList, EmfeMessage);
          this.$emit('exceededSize', file, this.fileList, EmfeMessage);
          return false;
        }
      }

      if (this.canUpload) {
        this.handleStart(file);
        this.beforeUpload(file, EmfeMessage);
        this.$emit('beforeUpload', file, EmfeMessage);
        this.canNotLoad();
        ajax({
          headers: this.headers,
          withCredentials: this.withCredentials,
          file,
          data: this.data,
          filename: this.name,
          action: this.action,
          onSuccess: (res) => {
            if (res.code === CONSTANT.AJAX_SUCCESS) {
              this.handleSuccess(res.data, file);
            } else {
              this.handleError('上传失败', res, file);
            }
          },
          onError: (err, response) => {
            this.handleError(err, response, file);
          },
        });
      }

      return false;
    },
    handleStart(file) {
      file.uid = Date.now() + this.tempIndex++;
      const fileData = {
        status: 'uploading',
        name: file.name,
        size: file.size,
        percentage: 0,
        uid: file.uid,
        showProgress: true,
      };
      this.fileList.push(fileData);
    },
    handleSuccess(res, file) {
      const fileData = this.getFile(file);

      if (fileData) {
        fileData.status = 'finished';
        fileData.response = res;
        if (this.fileType.indexOf('image') > -1) {
          this.loadImg(res.url, res, fileData);
        } else {
          this.canLoad();
          this.success(res, fileData, this.fileList, EmfeMessage);
          this.$emit('success', res, fileData, this.fileList, EmfeMessage);
        }
      }
    },
    handleError(err, response, file) {
      const fileData = this.getFile(file);
      const fileList = this.fileList;

      fileData.status = 'fail';
      this.canLoad();
      this.resetInputFile();

      fileList.splice(fileList.indexOf(fileData), 1);
      this.error(err, response, file, EmfeMessage);
      this.$emit('error', err, response, file, EmfeMessage);
    },
    getFile(file) {
      const fileList = this.fileList;
      let target;
      fileList.every((item) => {
        target = file.uid === item.uid ? item : null;
        return !target;
      });
      return target;
    },
    loadImg(src, res, fileData) {
      const img = new Image();
      img.src = src;
      img.onload = () => {
        this.src = src;
        setTimeout(this.setAlign.bind(this, res), 0);
        this.canLoad();
        this.success(res, fileData, this.fileList, EmfeMessage);
        this.$emit('success', res, fileData, this.fileList, EmfeMessage);
      };
    },
    closeFn() {
      this.closeCommon();
    },
    closePlusFn() {
      if (!this.disabled) {
        this.closeCommon();
      }
    },
    closeCommon() {
      this.src = '';
      this.canShow = false;
      this.resetInputFile();
      this.close(EmfeMessage);
      this.$emit('close', EmfeMessage);
    },
    resetInputFile() {
      if (this.type === 'icon') {
        this.$refs.upload.value = '';
      } else {
        this.$refs.uploadPlus.value = '';
      }
      this.dragWidth = 'auto';
      this.dragHeight = 400;
      this.initIntercept();
    },
    canLoad() {
      this.canUpload = true;
      if (this.type === 'icon') {
        this.iconText = this.buttonText;
      } else {
        this.plusText = '+';
      }
    },
    canNotLoad() {
      this.canUpload = false;
      if (this.type === 'icon') {
        this.iconText = '上传中';
      } else {
        this.plusText = '···';
      }
    },
  },
  watch: {
    url(val, oldVal) {
      if (val !== oldVal) {
        this.initImg(val);
      }
    },
  },
};
</script>
