<template>
  <div class="DialogDemoSettings-style">
    <h2>选择加载模型</h2>
    <el-radio-group v-model="cur_model_type">
      <el-radio :label="refer.SLOPE_NORMAL">平常状态模型</el-radio>
      <el-radio :label="refer.SLOPE_EARTHQUAKE">地震状态模型</el-radio>
    </el-radio-group>
    <p>{{model?(cur_model_type===init_model_type?'BP 神经网络模型已加载':'点击“确定”后将加载该模型'):'当前未加载神经网络模型'}}</p>
    <el-divider></el-divider>
    <h2>数据生成设置</h2>
    <div class="generate_action_bar">
      <div class="timer_switch">
        <span>定时生成</span>
        <el-switch v-model="is_timer_on" :disabled="!model"></el-switch>
      </div>
      <el-button type="text" @click="handleGenerate100" :disabled="!model">立刻生成 100 个数据</el-button>
    </div>
    <h3>位移数值范围（米）：</h3>
    <div class="input_with_label">
      <label for="range_min_input">最小值</label>
      <el-input type="number" v-model="range_min" id="range_min_input" class="input" @focus="handleFocus" />
    </div>
    <div class="input_with_label">
      <label for="range_max_input">最大值</label>
      <el-input type="number" v-model="range_max" id="range_max_input" class="input" @focus="handleFocus" />
    </div>
    <h3>定时生成间隔（秒）：</h3>
    <el-input type="number" v-model="cur_delay" id="delay_input" class="ainput" min="0.5" step="0.5"
              @focus="handleFocus" />

    <el-divider></el-divider>
    <h2>模型预测演示</h2>
    <p>请按顺序输入埋点 #1～#15 的位移值，数字之间用逗号分隔。</p>
    <el-input type="textarea" v-model="predict_input" placeholder="请注意位移的正负号，以远离边坡的方向为负值。" autosize></el-input>
    <div class="predict_action_bar">
      <el-button :disabled="!model" @click="handlePredictFos">预测</el-button>
      <span><b>预测强度折减系数：</b>{{predict_output}}</span>
    </div>
  </div>
</template>

<script>
  import { SLOPE_NORMAL, SLOPE_EARTHQUAKE } from '../../assets/refer.js';
  import { getPrediction } from '../../methods/MLFunction.js';
  import { getDataGenerator, generateManyData } from '../../methods/generateDemoData.js';
  import { getFs } from '../../methods/assistFunctions.js';
  import { setTimer, loadModel } from '../../store/effect.js';

  export default {
    data() {
      return {
        cur_delay: this.$store.state.timer.delay / 1000,
        range_min: 1.0e-4,
        range_max: 1.0e-3,
        is_timer_on: this.$store.state.timer.isOn,
        predict_input: '',
        predict_output: '待输入埋点位移',
        cur_model_type: this.$store.state.model_type,
        init_model_type: this.$store.state.model_type,
        refer: {
          SLOPE_NORMAL,
          SLOPE_EARTHQUAKE
        }
      };
    },
    computed: {
      model() {
        return this.$store.state.model;
      },
      range() {
        const input_range = [Number(this.range_min), Number(this.range_max)];
        return [Math.min(...input_range), Math.max(...input_range)]; // 防止有人把最大值和最小值输反
      },
      database() {
        return this.$store.state.database.get('1');
      }
    },
    methods: {
      handleFocus(e) {
        e.target.select();
      },
      handleGenerate100() {
        const fs = getFs(this.init_model_type, this.$store.state);
        generateManyData(this.database, this.range, fs, this.model, this.$store.commit);
      },
      async handlePredictFos() {
        const input_arr = this.predict_input.trim()
          .replace(/(,|，)$/, '')
          .split(/(?:,|，)\s*/)
          .map(item => Number(item));
        const model = this.model;
        if (input_arr.length !== 15) {
          console.warn('输入数据格式不符');
          return;
        }
        const output = await getPrediction(input_arr, model);
        this.predict_output = output;
      },
      handleSubmit() {
        const handleSetTimer = () => {
          const fs = getFs(this.cur_model_type, this.$store.state);
          const dataGenerator = getDataGenerator(this.database, this.range, fs, this.$store.state, this.$store.commit); // 由于传入函数的是当前组件的 model，为了使 model 能即时更新， 直接传 state 进去
          this.$store.commit(setTimer, {
            fn: dataGenerator,
            delay: this.cur_delay * 1000,
            isOn: this.is_timer_on
          });
        };

        if (this.cur_model_type !== this.init_model_type) { // 读取模型开销较大，所以先验证，没必要就不读
          this.$store.dispatch(loadModel, this.cur_model_type)
            .then(handleSetTimer);
        } else {
          handleSetTimer();
        }
      }
    },
    props: {
      submit_fn: {
        type: Function
      }
    },
    mounted() {
      this.$emit('update:submit_fn', this.handleSubmit);
    }
  };
</script>

<style lang="scss">
  .DialogDemoSettings-style {
    h2 {
      margin: 0 0 10px;
    }

    h3 {
      margin: 0 0 5px;
    }

    p {
      margin: 10px 0;
    }

    .input_with_label {
      margin-bottom: 10px;

      label {
        margin-right: 10px;
      }

      .input {
        width: 50%;
      }
    }

    .generate_action_bar {
      display: flex;
      justify-content: space-between;
      align-items: baseline;
      margin-bottom: 10px;

      .timer_switch {
        span {
          margin-right: 10px;
          font-weight: 700;
        }
      }
    }

    .predict_action_bar {
      display: flex;
      justify-content: space-between;
      align-items: baseline;
      margin-top: 10px;
    }
  }
</style>
