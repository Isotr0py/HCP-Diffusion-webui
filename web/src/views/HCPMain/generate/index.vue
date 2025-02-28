<template>
  <div class="txt-img-wrapper">
    <div class="row-top">
      <h-preview-img :imgs="imgs" :progress="progress"></h-preview-img>
      <div class="btn-and-config">
        <h-block class="outerWrapperShadow prompt">
          <el-input
            type="textarea"
            placeholder="Prompt"
            v-model="params.prompt"
            maxlength="1500"
            show-word-limit
            autofocus
            :autosize="{ minRows: 6, maxRows: 6 }"
          ></el-input>
          <el-input
            type="textarea"
            placeholder="Negative prompt"
            v-model="params.neg_prompt"
            maxlength="1500"
            show-word-limit
            :autosize="{ minRows: 6, maxRows: 6 }"
          ></el-input>
        </h-block>
      </div>
      <div class="btn-and-config">
        <h-block
          class="outerWrapperShadow"
          style="height: auto; flex: initial; padding: 10px 5px 10px"
        >
          <slot name="changeModel"></slot>
        </h-block>
        <h-block class="outerWrapperShadow" style="margin-bottom: 0; flex: initial">
          <div class="generate-box">
            <el-button
              class="primary-btn"
              type="primary"
              @click="handleGenerate"
              v-show="status !== STATUS.LOADING"
            >
              Generate
            </el-button>
            <div class="generate-box-btns" v-show="status === STATUS.LOADING">
              <!-- 中断 -->
              <el-button type="text" @click="handleInterrupt">Interrupt</el-button>
            </div>
          </div>
          <div class="config-row">
            <el-checkbox v-model="params.save.save_cfg">save_cfg</el-checkbox>
          </div>
          <div class="config-row">
            <HConfigSelect
              tooltip="generate.save.image_type"
              label="image_type"
              :options="image_typ_options"
              v-model="params.save.image_type"
              style="font-size: 13px"
            />
            <HConfigInputNumber
              label="quality"
              tooltip="generate.save.quality"
              :min="1"
              :max="100"
              :step="1"
              v-model="params.save.quality"
              style="font-size: 13px"
            />
          </div>
        </h-block>
      </div>
    </div>
    <div class="row-bottom">
      <div class="config-main">
        <div class="config-item">
          <div class="config-item-scroll-wrap">
            <!-- 取样方法配置 -->
            <OtherConfig ref="otherConfig" :params="params" @updateConfig="onUpdateConfig" />
            <!-- infer_args -->
            <InferArgsConfig
              ref="inferArgsConfig"
              :params="params"
              @updateConfig="onUpdateConfig"
            />
          </div>
        </div>
        <div class="config-item">
          <div class="config-item-scroll-wrap">
            <!-- offload -->
            <OffloadConfig ref="offloadConfig" :params="params" @updateConfig="onUpdateConfig" />

            <!-- condition -->
            <ConditionConfig
              ref="conditionConfig"
              :params="params"
              @updateConfig="onUpdateConfig"
              @onOpen="changeOpenConditionConfig"
            />

            <!-- ex_input -->
            <EXInputConfig
              ref="exInputConfig"
              :params="params"
              @updateConfig="onUpdateConfig"
              @onOpen="changeOpenEXInputConfig"
            />

            <!-- new_components -->
            <NewComponentsConfig
              ref="newComponentsConfig"
              :params="params"
              :pretrained_model_name_or_path_options="pretrained_model_name_or_path_options"
              @updateConfig="onUpdateConfig"
            />
          </div>
        </div>
        <div class="config-item">
          <div class="config-item-scroll-wrap">
            <!-- merge -->
            <MergeConfig
              ref="mergeConfig"
              :params="params"
              :merge_group_lora_path="merge_group_lora_path"
              :merge_group_part_path="merge_group_part_path"
              :merge_group_plugin_controlnet1_path="merge_group_plugin_controlnet1_path"
              @refresh="handleRefresh"
              @updateConfig="onUpdateConfig"
              @onOpen="changeOpenMergeConfig"
            />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import { default_data, image_typ_options, STATUS_TYPE } from '@/constants/index';
import {
  generateAction,
  getGenerateProgress,
  stopGenerate,
  getGenerateConfig
} from '@/api/generate';
import OtherConfig from './components/otherConfig.vue';
import ConditionConfig from './components/conditionConfig.vue';
import EXInputConfig from './components/exInputConfig.vue';
import OffloadConfig from './components/offloadConfig.vue';
import InferArgsConfig from './components/inferArgsConfig.vue';
import NewComponentsConfig from './components/newComponentsConfig.vue';
import MergeConfig from './components/mergeConfig.vue';
import { getGenerateDir } from '@/api/file';
import { handleOptions, validateParams } from '@/utils/index';
import { merge, isEmpty } from 'lodash-es';
import { mapGetters } from 'vuex';
const STATUS = {
  LOADING: 'loading',
  SUCCESS: 'success'
};

export default {
  name: 'TxtImg',
  components: {
    OtherConfig,
    ConditionConfig,
    EXInputConfig,
    OffloadConfig,
    InferArgsConfig,
    NewComponentsConfig,
    MergeConfig
  },
  props: {
    pretrained_model: {
      type: String,
      default: ''
    },
    yaml_template_sn: {
      type: String,
      default: ''
    }
  },
  data() {
    return {
      status: STATUS.SUCCESS,
      STATUS,
      params: JSON.parse(JSON.stringify(default_data)),
      image_typ_options,
      merge_group_lora_path: [],
      merge_group_part_path: [],
      merge_group_plugin_controlnet1_path: [],
      pretrained_model_name_or_path_options: [],
      imgs: [],
      progress: 0,
      timer: null,
      isOpenMergeConfig: false,
      isOpenConditionConfig: false,
      isOpenEXInput: false
    };
  },
  computed: {
    ...mapGetters(['generateSn'])
  },
  watch: {
    pretrained_model: {
      handler: function (val) {
        this.params.pretrained_model = val;
      },
      immediate: true
    }
  },
  created() {
    this.initDefaultData();
  },
  methods: {
    async handleGenerate() {
      if (!this.validate()) return;
      this.status = 'loading';
      this.imgs = [];
      const info = this.handlerInfo();
      const result = await generateAction({
        info: { ...info, pretrained_model: this.pretrained_model }
      }).catch((err) => {
        this.$message.error(err || 'generate failed');
        this.status = STATUS.SUCCESS;
      });
      if (!result) return;
      const { sn } = result;
      this.$store.commit('setGenerateSnSn', sn);
      this.getProgress();
    },
    // 中断
    async handleInterrupt() {
      const result = await stopGenerate(this.generateSn).catch(() => {
        this.$message.error('interrupt failure');
      });
      if (!result) return;
      const { status } = result;
      this.status = STATUS.SUCCESS;
      this.progress = 0;
      this.timer && clearTimeout(this.timer);
      // 清空
      this.imgs = [];
      this.$message.success('interrupt success');
      if (status !== STATUS_TYPE.ACTIVE_INTERRUPT) {
        this.$emit('onMessage', status);
      }
    },
    // 轮询获取进度
    async getProgress() {
      const result = await getGenerateProgress(this.generateSn).catch(() => {
        this.$message.error('fetch generate progress failed');
      });
      if (!result) return;
      const { progress, sn, images = [], status } = result;
      // 定时轮询 1s
      if (progress < 100) {
        this.timer = setTimeout(() => this.getProgress(), 2000);
        this.progress = progress;
      } else {
        if (status === STATUS_TYPE.END) {
          this.$message.success('generate succeeded');
        }
        this.status = STATUS.SUCCESS;
        this.imgs = images;
        this.progress = 0;
        this.$store.commit('setGenerateSnSn', sn);
      }
      this.$emit('onMessage', status);
    },
    // 重新获取options
    async handleRefresh(field) {
      const result = await getGenerateDir({ path: field }).catch(() => {
        this.$message.error(`fetch setting ${field} failed`);
      });
      if (!result) return;
      const { files = [] } = result;
      this[field] = handleOptions(files);
    },

    async initDefaultData() {
      const result = await getGenerateConfig(this.generateSn).catch(() => {
        this.$message.error('fetch setting failed');
      });
      if (!result) return;
      const {
        info,
        is_pending,
        progress,
        images = [],
        merge_group_lora_path = [],
        merge_group_part_path = [],
        merge_group_plugin_controlnet1_path = [],
        pretrained_mode = [],
        server_yaml_file = [],
        pretrained_model_name_or_path = []
      } = result;
      const newInfo = JSON.parse(JSON.stringify(info));
      this.$set(this, 'params', JSON.parse(JSON.stringify(default_data)));
      this.$set(this, 'params', merge(this.params, info));
      if (!isEmpty(newInfo)) {
        setTimeout(() => {
          this.initAllConfig(newInfo);
        });
      }
      if (images.length) {
        this.imgs = images;
      }
      if (is_pending) {
        this.status = STATUS.LOADING;
        this.progress = progress;
        this.getProgress();
      }
      this.merge_group_lora_path = handleOptions(merge_group_lora_path);
      this.merge_group_part_path = handleOptions(merge_group_part_path);
      this.merge_group_plugin_controlnet1_path = handleOptions(merge_group_plugin_controlnet1_path);
      this.pretrained_model_name_or_path_options = handleOptions(pretrained_model_name_or_path);
      const { pretrained_model } = info || {};
      this.$emit('getPretrainedMode', {
        options: handleOptions(pretrained_mode),
        pretrained_model,
        server_yaml_file,
        files: 'generate_server_yaml_file_options',
        valueFiles: 'generate_yaml_template_sn'
      });
    },

    initAllConfig(info) {
      this.$refs.mergeConfig && this.$refs.mergeConfig.initConfig(info);
      this.$refs.conditionConfig && this.$refs.conditionConfig.initConfig(info);
      this.$refs.exInputConfig && this.$refs.exInputConfig.initConfig(info);
      this.$refs.offloadConfig && this.$refs.offloadConfig.initConfig(info);
      this.$refs.newComponentsConfig && this.$refs.newComponentsConfig.initConfig(info);
      this.$refs.inferArgsConfig && this.$refs.inferArgsConfig.initConfig(info);
      this.$refs.otherConfig && this.$refs.otherConfig.initConfig(info);

      const { merge } = info;
      if (merge) {
        this.isOpenMergeConfig = true;
      }
    },

    onUpdateConfig({ value, field }) {
      if (field === 'other') {
        this.params = { ...this.params, ...value };
      } else {
        this.params[field] = value || null;

        if (field === 'condition') {
          this.$refs.inferArgsConfig && this.$refs.inferArgsConfig.initStrength();
        }
      }
    },

    handlerInfo() {
      const params = JSON.parse(JSON.stringify(this.params));
      const { isOpenMergeConfig } = this;
      const { infer_args, new_components, merge } = params;
      if (!isOpenMergeConfig) {
        params.merge = null;
      }
      if (infer_args && [null, undefined, ''].includes(infer_args.strength)) {
        delete params.infer_args.strength;
      }
      if (new_components) {
        if ('vae' in new_components && [null, undefined, ''].includes(new_components.vae)) {
          delete params.new_components.vae;
        }
        // 支持不同参数的scheduler
        if (
          'lower_order_final' in new_components.scheduler &&
          [null, undefined, ''].includes(new_components.scheduler.lower_order_final)
        ) {
          delete params.new_components.scheduler.lower_order_final;
        }
        if (
          'algorithm_type' in new_components.scheduler &&
          [null, undefined, ''].includes(new_components.scheduler.algorithm_type)
        ) {
          delete params.new_components.scheduler.algorithm_type;
        }
        if (
          'use_karras_sigmas' in new_components.scheduler &&
          [null, undefined, ''].includes(new_components.scheduler.use_karras_sigmas)
        ) {
          delete params.new_components.scheduler.use_karras_sigmas;
        }
      }

      if (merge) {
        Object.keys(merge).forEach((key) => {
          // 判断plugin是否为空数组
          if (merge[key] && merge[key].plugin && !merge[key].plugin.length) {
            merge[key].plugin = null;
          }
        });
      }

      return params;
    },

    changeOpenMergeConfig(e) {
      this.isOpenMergeConfig = e;
    },
    changeOpenConditionConfig(e) {
      this.isOpenConditionConfig = e;
    },
    changeOpenEXInputConfig(e) {
      this.isOpenEXInput = e;
    },
    // 校验参数
    validate() {
      const self = this;
      const params = JSON.parse(JSON.stringify(this.params));
      const requiredField = {
        'condition.image': {
          // 可自定义
          validate: (val) => {
            if (!self.isOpenConditionConfig) return true;
            return !['', undefined, null].includes(val);
          },
          message: 'condition image is required'
        },
        'ex_input.cond.image': {
          // 可自定义
          validate: (val) => {
            if (!self.isOpenEXInput) return true;
            return !['', undefined, null].includes(val);
          },
          message: 'ex_input image is required'
        },
        'merge.*.lora.*.path': {
          validate: (val) => {
            if (!self.isOpenMergeConfig) return true;
            return !['', undefined, null].includes(val);
          },
          message: 'merge.*.lora.*.path is required'
        },
        'merge.*.part.*.path': {
          validate: (val) => {
            if (!self.isOpenMergeConfig) return true;
            return !['', undefined, null].includes(val);
          },
          message: 'merge.*.part.*.path is required'
        },
        'merge.*.plugin.*.path': {
          validate: (val) => {
            if (!self.isOpenMergeConfig) return true;
            return !['', undefined, null].includes(val);
          },
          message: 'merge.*.plugin.*.path is required'
        },
        'new_components.vae._target_': {
          message: 'new_components.vae._target_ is required'
        }
      };
      return validateParams(requiredField, params);
    }
  },
  beforeDestroy() {
    clearTimeout(this.timer);
  }
};
</script>
<style lang="scss" scoped>
.outerWrapperShadow {
  &.prompt {
    margin-bottom: 0;
    @include flexLayout(column, 20px, space-between, center);
  }
}

.txt-img-wrapper {
  width: 100%;
  height: 100%;
  @include flexLayout(column, 0);
  .row-top {
    width: 100%;
    height: 340px;
    @include flexLayout(row, 20px);
    padding: 20px 20px 14px;
    box-sizing: border-box;
    .btn-and-config {
      flex: 3;
      height: 100%;
      background-color: #fff;
      @include border-radius();
      @include flexLayout(column);
      justify-content: space-between;
      &.prompt {
        margin-bottom: 0;
      }
      .generate-box {
        height: 72px;
        &-btns {
          display: flex;
          height: 100%;
          .el-button {
            flex: 1;
            height: 100%;
            border: 1px solid #b9c0cb;
            color: #3b414f;
            background: #b9c0cb;
            @include blockShadow();
            &:hover {
              background: #a6abb3;
              border: 1px solid #a6abb3;
            }
          }
          & > :nth-of-type(1) {
            border-radius: 4px 0 0 4px;
          }
        }
      }
    }
  }
  .row-bottom {
    flex: 1;
    overflow-y: scroll;
    padding: 4px 0px 10px 0px;
    overflow: hidden;
    .config-main {
      height: 100%;
      @include flexLayout(row, 0);
      overflow: hidden;
      .config-item {
        flex: 1;
        width: 100%;
        overflow-y: scroll;
        padding: 10px 20px;
        box-sizing: border-box;
        &::-webkit-scrollbar {
          width: 5px;
          height: 5px;
          background-color: transparent;
        }
        // 滚动条轨道
        &::-webkit-scrollbar-track {
          // box-shadow: inset 0 0 6px rgba(0, 0, 0, 0.1);
          background-color: transparent;
        }
        &-scroll-wrap {
          width: 100%;
          @include flexLayout(column, 20px);
        }
      }
    }
  }
}
</style>
