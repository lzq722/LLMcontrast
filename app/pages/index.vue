<template>
  <div class="p-10">
    <h1 class="text-3xl font-bold text-center text-gray-100 mb-8 tracking-wide">大模型智能分析与对比平台</h1>

    <div class="mb-8 p-6 bg-gray-800 rounded-lg shadow-lg">
      <h2 class="text-2xl font-bold text-center text-gray-100 mb-4">
        智能模型分析
      </h2>
      <p class="text-center text-gray-400 mb-6">
        输入一条评论，后端将自动分类并选择最优模型进行分析。
      </p>
      <div class="flex gap-4">
        <el-input
          v-model="singleComment"
          placeholder="请输入需要分析的评论..."
          size="large"
          clearable
          @keyup.enter="analyzeSingleComment"
        />
        <el-button
          type="success"
          @click="analyzeSingleComment"
          :loading="isAnalyzing"
          size="large"
        >
          分析评论
        </el-button>
      </div>

      <div v-if="analysisResult" class="mt-6 p-4 bg-gray-700 rounded-md">
        <p class="text-gray-300 mb-2">
          <span class="font-bold">接收到的评论:</span> {{ analysisResult.comment_received }}
        </p>
        <p class="text-gray-300 mb-2">
          <span class="font-bold">智能分类标签:</span>
          <el-tag type="info" size="small" class="ml-2">{{ analysisResult.classified_tag || 'N/A' }}</el-tag>
        </p>
        <p class="text-gray-300 mb-2">
          <span class="font-bold">最终分析模型:</span>
          <el-tag type="primary" size="small" class="ml-2">{{ analysisResult.model_used_for_analysis || 'N/A' }}</el-tag>
        </p>
        <p class="text-gray-300 mb-2">
          <span class="font-bold">模型回复:</span> {{ analysisResult.final_reply || '无回复' }}
        </p>
        <p v-if="analysisResult.error" class="text-red-400">
          <span class="font-bold">错误:</span> {{ analysisResult.error }}
        </p>
      </div>
    </div>
    <hr class="my-10 border-gray-600" />

    <h2 class="text-2xl font-bold text-center text-gray-100 mb-8">批量模型对比测试</h2>
    <div class="flex justify-center items-center">
      <el-checkbox-group
        v-model="selectedLLMs"
        @change="handleCheckedCitiesChange"
      >
        <el-checkbox
          v-for="item in LLMlist"
          :key="item"
          :value="item"
          size="large"
          class="bg-gray-700 rounded-lg px-6 py-3 shadow transition hover:shadow-xl text-gray-200"
        >
          {{ item }}
        </el-checkbox>
      </el-checkbox-group>
    </div>
    <div class="text-center mt-4">
        <el-button type="primary" @click="test">进行对比</el-button>
    </div>


    <div v-if="Object.keys(results).length > 0" class="mt-8">
      <h2 class="text-2xl font-bold text-gray-100 mb-4">对比结果展示</h2>
      <div v-for="(llm, llmIndex) in Object.keys(results)" :key="llmIndex" class="mb-4">
        <h3 class="text-xl font-bold text-gray-200 mb-2">{{ llm }}</h3>
        <div class="flex flex-wrap gap-4 max-h-64 overflow-y-auto">
          <div
            v-for="(item, itemIndex) in Object.keys(results[llm] || {})"
            :key="itemIndex"
            :class="[
              'p-4 rounded-lg mb-2 break-words w-full sm:w-32 hover:shadow-xl transition cursor-pointer',
              results[llm]?.[Number(item)]?.match === 1
              ? 'bg-green-500'
              : results[llm]?.[Number(item)]?.match === 0
                ? 'bg-red-500'
                : 'bg-gray-800'
            ]"
            @click="questionIdx = item"
          >
            <p class="text-gray-300 break-words text-center">{{ item }}</p>
          </div>
        </div>
      </div>
      <div v-if="questionIdx" class="mt-4">
        <h4 class="text-lg font-bold text-gray-200 mb-2">详细信息</h4>
        <p class="text-gray-300 mb-2">评论文本: {{ questioncontent }}</p>
        <div v-for="llm in Object.keys(results)" :key="llm" class="mb-2">
          <p class="text-gray-300">
            <span class="font-bold">{{ llm }}:</span>
            {{ results[llm]?.[Number(questionIdx)]?.reply || '无回复' }}
          </p>
        </div>
        <p class="text-gray-300">正确倾向: {{ questionpicking }}</p>
      </div>

      <div class="mt-8">
        <h2 class="text-2xl font-bold text-gray-100 mb-4">性能评估图表</h2>
        <div class="flex flex-wrap gap-8 mb-4 justify-center">
          <div
            v-for="llm in Object.keys(evaluation)"
            :key="llm"
            class="bg-gray-700 rounded-lg px-6 py-4 shadow text-gray-200 flex flex-col items-center"
          >
            <span class="font-bold text-lg mb-2">{{ llm }}</span>
            <span>平均响应时间: {{ evaluation[llm]?.time ?? 0 }} ms</span>
          </div>
        </div>
        <div ref="chartRef" class="w-full h-136" />
      </div>
    </div>
    </div>
</template>

<script setup lang="ts">
  import type { CheckboxValueType } from 'element-plus'
  import * as echarts from 'echarts';
  import { ref, watch } from 'vue';

  const singleComment = ref('');
  const isAnalyzing = ref(false);
  const analysisResult = ref<Record<string, any> | null>(null);

  const analyzeSingleComment = async () => {
    if (!singleComment.value.trim()) {
      alert('评论内容不能为空！');
      return;
    }

    isAnalyzing.value = true;
    analysisResult.value = null;

    try {
      // URL points to the new intelligent analysis endpoint
      const response = await fetch('http://127.0.0.1:5001/api/intelligent_analysis', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          comment: singleComment.value,
        }),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error || `HTTP error! status: ${response.status}`);
      }

      const result = await response.json();
      analysisResult.value = result;
      console.log('Intelligent API Response:', result);

    } catch (error) {
      console.error('Error calling the intelligent API:', error);
      analysisResult.value = {
        error: (error as Error).message,
        comment_received: singleComment.value,
      };
    } finally {
      isAnalyzing.value = false;
    }
  };


  type AnswerInfo = {
    reply: string
    solution: string
    match: number
    time: number
  }
  type answer = {
    accuracy: number
    precision_pos: number
    precision_neg: number
    recall_pos: number
    recall_neg: number
    F1_pos: number
    F1_neg: number
    time: number
  }
  const questionIdx = ref("")
  const questioncontent = ref("")
  const questionpicking = ref("")
  const chartRef = ref<HTMLElement | null>(null);
  let chartInstance: echarts.ECharts | null = null;
  const picking = [ "neg", "pos" ]
  const modellist = [
    { name: 'DeepSeek-V3', value: 'deepseek-ai/DeepSeek-V3' },
    { name: 'DeepSeek-R1', value: 'deepseek-ai/DeepSeek-R1' },
    { name: 'ChatGLM', value: 'THUDM/GLM-Z1-32B-0414' },
    { name: 'Qwen', value: 'Qwen/Qwen3-30B-A3B' },
  ]
  const selectedLLMs = ref([])
  const results = ref<{ [llm: string]: { [questionIdx: number]: AnswerInfo } }>({})
  const evaluation = ref<{ [llm: string]: answer}>({})
  const LLMlist = ref([
    'DeepSeek-V3',
    'DeepSeek-R1',
    'ChatGLM',
    'Qwen',
  ])
  const options = {
    method: 'POST',
    headers: {Authorization: 'Bearer sk-jjzrcpevjqfygoruzylqssgudjgxiguwisnsqlbwrwcfhxsm', 'Content-Type': 'application/json'},
      body: JSON.stringify({
      model: "",
      messages: [
        {
        role: "user",
        content: ""
        }
      ],
      stream: false,
      max_tokens: 512,
      min_p: 0.05,
      stop: null,
      temperature: 0.7,
      top_p: 0.7,
      top_k: 50,
      frequency_penalty: 0.5,
      n: 1,
      response_format: { type: "text" },
      })
  }
  const handleCheckedCitiesChange = (value: CheckboxValueType[]) => {
    console.log('Selected LLMs:', value)
  }
  const test = async () => {
    questionIdx.value = "";
    results.value = {};
    const randomNumbers: number[] = Array.from({ length: 10 }, () => Math.floor(Math.random() * 1000));
    const half = Math.floor(randomNumbers.length / 2);
    const negnum: number[] = randomNumbers.slice(0, half);
    const posnum: number[] = randomNumbers.slice(half);

    selectedLLMs.value.forEach(llm => {
      if (!results.value[llm]) {
        results.value[llm] = {};
      }
      picking.forEach(p => {
        const randomNumbers = p === "neg" ? negnum : posnum;
        for (const i in randomNumbers) {
          fetch(`/2000/${p}/${p}.${randomNumbers[i]}.txt`)
          .then(response => response.arrayBuffer())
          .then(async buffer => {
            const decoder = new TextDecoder('gb2312');
            const text = decoder.decode(buffer);
            const prompt = `
            请判断以下这段评论的情感倾向, 不论分了几点或是几段, 都为一整条评论, 只需要回答一次：
            ${text}
            选项：[正面, 负面]
            请直接回答分类结果, 并且只可以回答所提供的选项中的内容。请仔细阅读文字中的每一个字, 以确保你能理解评论的情感倾向, 尽量避免误判。
            `;
            options.body = JSON.stringify({
              ...JSON.parse(options.body),
              model: modellist.find(m => m.name === llm)?.value,
              messages: [
                {
                  role: "user",
                  content: prompt
                }
              ]
            });
            const startTime = performance.now();
            await fetch('https://api.siliconflow.cn/v1/chat/completions', options)
            .then(response => response.json())
            .then(response => {
              const endTime = performance.now();
              const consume = endTime - startTime;
              results.value[llm] = results.value[llm] || {};
              const num = randomNumbers[i];
              if (typeof num === 'number' && !isNaN(num)) {
                results.value[llm][num] = {
                  reply: response.choices[0].message.content,
                  solution: p === "neg" ? "负面" : "正面",
                  match: response.choices[0].message.content.includes(p === "neg" ? "负面" : "正面") ? 1 : 0,
                  time: consume
                };
              }
            })
            .catch(err => console.error(err));
          })
          .catch(error => {
            console.error(`Error reading ${p}.${randomNumbers[i]}.txt:`, error);
          });
        }
      })
    });
    await Promise.all(
      selectedLLMs.value.flatMap(llm =>
        picking.flatMap(p => {
          const randomNumbers = p === "neg" ? negnum : posnum;
          return randomNumbers.map(num =>
            new Promise<void>((resolve) => {
              const check = () => {
                if (results.value[llm]?.[num]) resolve();
                else setTimeout(check, 100);
              };
              check();
            })
          );
        })
      )
    );
    calcEvaluate();
  }
  const calcEvaluate = () => {
    evaluation.value = {};
    Object.keys(results.value).forEach(llm => {
      const answers = Object.values(results.value[llm] || {});
      if (answers.length === 0) return;
      const correct = answers.filter(a => a.match === 1).length;
      const total = answers.length;
      const avgTime = answers.reduce((sum, a) => sum + a.time, 0) / total;
      const TP = answers.filter(a => a.match === 1 && a.solution === "正面").length;
      const TN = answers.filter(a => a.match === 1 && a.solution === "负面").length;
      const FP = answers.filter(a => a.match === 0 && a.solution === "正面").length;
      const FN = answers.filter(a => a.match === 0 && a.solution === "负面").length;
      evaluation.value[llm] = {
        accuracy: total > 0 ? +(correct / total).toFixed(2) : 0,
        precision_pos: (TP + FP) > 0 ? +(TP / (TP + FP)).toFixed(2) : 0,
        precision_neg: (TN + FN) > 0 ? +(TN / (TN + FN)).toFixed(2) : 0,
        recall_pos: (TP + FN) > 0 ? +(TP / (TP + FN)).toFixed(2) : 0,
        recall_neg: (TN + FP) > 0 ? +(TN / (TN + FP)).toFixed(2) : 0,
        F1_pos: ((TP / (TP + FP)) + (TP / (TP + FN))) > 0 ? +(2 * (TP / (TP + FP)) * (TP / (TP + FN)) / ((TP / (TP + FP)) + (TP / (TP + FN)))).toFixed(2) : 0,
        F1_neg: ((TN / (TN + FP)) + (TN / (TN + FN))) > 0 ? +(2 * (TN / (TN + FP)) * (TN / (TN + FN)) / ((TN / (TN + FP)) + (TN / (TN + FN)))).toFixed(2) : 0,
        time: +avgTime.toFixed(2)
      };
    });
  };
  watch(results, () => {
    calcEvaluate();
  });
  watch(evaluation, () => {
    renderChart();
  }, { deep: true });
  const renderChart = () => {
    if (!chartRef.value) return;
    if (!chartInstance) {
      chartInstance = echarts.init(chartRef.value);
    }
    const propertys = ['正确率', '正面精确率', '负面精确率', '正面召回率', '负面召回率', '正面F1值', '负面F1值'];
    const answerKeys = ['accuracy', 'precision_pos', 'precision_neg', 'recall_pos', 'recall_neg', 'F1_pos', 'F1_neg'];
    const colors = ['#4E79A7', '#F28E2B', '#E15759', '#76B7B2', '#59A14F', '#EDC948', '#B07AA1', 'FF9DA7'];
    const labels = Object.keys(evaluation.value);

    const series = labels.map(model => ({
      name: model,
      type: 'bar',
      data: answerKeys.map(key => {
        const value = evaluation.value[model]?.[key as keyof answer];
        return ((value ?? 0) * 100).toFixed(2);
      }),
      itemStyle: { color: colors[labels.indexOf(model)] },
    }))

    const option = {
      tooltip: { trigger: 'axis' },
      legend: { data: labels, textStyle: { color: '#ccc' } },
      xAxis: { type: 'category', data: propertys, axisLabel: { color: '#ccc' } },
      yAxis: [{ type: 'value', name: '百分比(%)', position: 'left', axisLabel: { color: '#ccc' }, min: 0, max: 100 }],
      series
    };
    chartInstance.setOption(option);
  };
  watch(questionIdx, (newVal) => {
    if (newVal) {
      const llm = Object.keys(results.value)[0];
      if (llm !== undefined) {
        questionpicking.value = (results.value[llm]?.[Number(newVal)]?.solution === "正面" ? "pos" : "neg");
        if (questionpicking.value) {
          fetch(`/2000/${questionpicking.value}/${questionpicking.value}.${questionIdx.value}.txt`)
          .then(response => response.arrayBuffer())
          .then(async buffer => {
            const decoder = new TextDecoder('gb2312');
            questioncontent.value = decoder.decode(buffer);
            questionpicking.value = (questionpicking.value === "pos" ? "正面" : "负面")
          })
          .catch(error => {
            console.error(`Error reading ${questionpicking.value}.${questionIdx.value}.txt:`, error);
          });
        }
      }
    }
  });
</script>

<style>
</style>
