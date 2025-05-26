<template>
  <div class="p-10">
    <h1 class="text-3xl font-bold text-center text-gray-100 mb-8 tracking-wide">选择大模型</h1>
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
    <el-button type="primary" @click="test">进行对比</el-button>
    <div v-if="Object.keys(results).length > 0" class="mt-8">
      <h2 class="text-2xl font-bold text-gray-100 mb-4">结果展示</h2>
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
        <!-- <p class="text-gray-300">模型回复: {{ results[llm]?.[Number(questionIdx)]?.reply }}</p> -->
        <p class="text-gray-300">正确倾向: {{ questionpicking }}</p>
        <!-- <p class="text-gray-300">模型: {{ llm }}</p> -->
      </div>
      <!-- <div v-if="Object.keys(evaluation).length > 0" class="mt-8"> -->
        <!-- <h2 class="text-2xl font-bold text-gray-100 mb-4">评估结果</h2> -->
        <!-- <div v-for="(llm, llmIndex) in Object.keys(evaluation)" :key="llmIndex" class="mb-4"> -->
          <!-- <h3 class="text-xl font-bold text-gray-200 mb-2">{{ llm }}</h3> -->
          <!-- <p class="text-gray-300">准确率: {{ (evaluation[llm]?.accuracy ?? 0) * 100 }}%</p> -->
          <!-- <p class="text-gray-300">平均响应时间: {{ evaluation[llm]?.time ?? 0 }} ms</p> -->
        <!-- </div> -->
      <!-- </div> -->
      <div class="mt-8">
        <h2 class="text-2xl font-bold text-gray-100 mb-4">评估结果</h2>
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
  // const finish = ref(false)
  const chartRef = ref<HTMLElement | null>(null);
  let chartInstance: echarts.ECharts | null = null;
  const picking = [ "neg", "pos" ]
  const modellist = [
    { name: 'DeepSeek-V3', value: 'deepseek-ai/DeepSeek-V3' },
    { name: 'DeepSeek-R1', value: 'deepseek-ai/DeepSeek-R1' },
    { name: 'Qwen', value: 'Qwen/Qwen3-30B-A3B' },
  ]
  const selectedLLMs = ref([])
  const results = ref<{ [llm: string]: { [questionIdx: number]: AnswerInfo } }>({})
  const evaluation = ref<{ [llm: string]: answer}>({})
  const LLMlist = ref([
    'DeepSeek-V3',
    'DeepSeek-R1',
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
      enable_thinking: false,
      // thinking_budget: 4096,
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
    console.log(results.value)
    console.log(questionIdx.value)
  }
  const test = async () => {
    results.value = {};
    const randomNumbers: number[] = Array.from({ length: 200 }, () => Math.floor(Math.random() * 1000));
    const half = Math.floor(randomNumbers.length / 2);
    const negnum: number[] = randomNumbers.slice(0, half);
    const posnum: number[] = randomNumbers.slice(half);
    console.log('Random Numbers:', randomNumbers);
    // 根据 selectedLLMs 初始化 results
    selectedLLMs.value.forEach(llm => {
      if (!results.value[llm]) {
      results.value[llm] = {};
      }
      picking.forEach(p => {
        console.log('picking:', p);
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
            // console.log('Prompt:', prompt);
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
              // console.log(response.choices[0].message.content)
              results.value[llm] = results.value[llm] || {};
              const num = randomNumbers[i];
              if (typeof num === 'number' && !isNaN(num)) {
                results.value[llm][num] = {
                  reply: response.choices[0].message.content,
                  solution: p === "neg" ? "负面" : "正面",
                  match: response.choices[0].message.content === (p === "neg" ? "负面" : "正面") ? 1 : 0,
                  time: consume
                };
              }
            })
            .catch(err => console.error(err));
          })
          .catch(error => {
            console.error(`Error reading neg.${randomNumbers[i]}.txt:`, error);
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
    console.log('All answers completed:', results.value);
    // 计算准确率和平均时间
    calcEvaluate();
    // console.log('Results:', results.value);
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
      console.log('TP:', TP, 'TN:', TN, 'FP:', FP, 'FN:', FN);
      evaluation.value[llm] = {
        // TP: answers.filter(a => a.match === 1 && a.solution === "正面").length,
        // TN: answers.filter(a => a.match === 1 && a.solution === "负面").length,
        // FP: answers.filter(a => a.match === 0 && a.solution === "正面").length,
        // FN: answers.filter(a => a.match === 0 && a.solution === "负面").length,
        accuracy: +(correct / total).toFixed(2),
        precision_pos: +(TP / (TP + FP)).toFixed(2),
        precision_neg: +(TN / (TN + FN)).toFixed(2),
        recall_pos: +(TP / (TP + FN)).toFixed(2),
        recall_neg: +(TN / (TN + FP)).toFixed(2),
        F1_pos: +(2 * (TP / (TP + FP)) * (TP / (TP + FN)) / ((TP / (TP + FP)) + (TP / (TP + FN)))).toFixed(2),
        F1_neg: +(2 * (TN / (TN + FP)) * (TN / (TN + FN)) / ((TN / (TN + FP)) + (TN / (TN + FN)))).toFixed(2),
        time: +avgTime.toFixed(2)
      };
    });
    console.log('Evaluation:', evaluation.value);
  };
  watch(results, () => {
    calcEvaluate();
  });
  watch(evaluation, () => {
    renderChart();
  }, { deep: true });
  const renderChart = () => {
    if (!chartRef.value) return;
    console.log('Rendering chart...');
    if (!chartInstance) {
      chartInstance = echarts.init(chartRef.value);
    }

    const propertys = ['正确率', '正面精确率', '负面精确率', '正面召回率', '负面召回率', '正面F1值', '负面F1值'];
    const answer = ['accuracy', 'precision_pos', 'precision_neg', 'recall_pos', 'recall_neg', 'F1_pos', 'F1_neg'];
    const colors = ['#4E79A7', '#F28E2B', '#E15759', '#76B7B2', '#59A14F', '#EDC948', '#B07AA1', 'FF9DA7'];
    const labels = Object.keys(evaluation.value);
    // const accuracyData = labels.map(llm => +(((evaluation.value[llm]?.accuracy ?? 0) * 100).toFixed(2)));
    // const precision_posData = labels.map(llm => +(((evaluation.value[llm]?.precision_pos ?? 0) * 100).toFixed(2)));
    // const precision_negData = labels.map(llm => +(((evaluation.value[llm]?.precision_neg ?? 0) * 100).toFixed(2)));
    // const recall_posData = labels.map(llm => +(((evaluation.value[llm]?.recall_pos ?? 0) * 100).toFixed(2)));
    // const recall_negData = labels.map(llm => +(((evaluation.value[llm]?.recall_neg ?? 0) * 100).toFixed(2)));
    // const F1_posData = labels.map(llm => +(((evaluation.value[llm]?.F1_pos ?? 0) * 100).toFixed(2)));
    // const F1_negData = labels.map(llm => +(((evaluation.value[llm]?.F1_neg ?? 0) * 100).toFixed(2)));
    // const timeData = labels.map(llm => +(evaluation.value[llm]?.time ?? 0).toFixed(2));

    const series = labels.map(model => ({
      name: model,
      type: 'bar',
      data: answer.map(a => {
        const value = evaluation.value[model]?.[a as keyof answer];
        return ((value ?? 0) * 100).toFixed(2);
      }),
      // yAxisIndex: (),
      itemStyle: { color: colors[labels.indexOf(model)] },
    }))

    const option = {
      tooltip: {
        trigger: 'axis',
      },
      legend: {
        data: labels,
        textStyle: { color: '#ccc' }
      },
      xAxis: {
        type: 'category',
        data: propertys,
        axisLabel: { color: '#ccc' }
      },
      yAxis: [
        {
          type: 'value',
          name: '百分比(%)',
          position: 'left',
          axisLabel: { color: '#ccc' },
          min: 0,
          max: 100
        }
      ],
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
          console.log('Selected item:', questionpicking.value);
        }
        fetch(`/2000/${questionpicking.value}/${questionpicking.value}.${questionIdx.value}.txt`)
        .then(response => response.arrayBuffer())
        .then(async buffer => {
          const decoder = new TextDecoder('gb2312');
          questioncontent.value = decoder.decode(buffer);
          questionpicking.value = (questionpicking.value === "pos" ? "正面" : "负面")
          // console.log('Question content:', questioncontent.value);
        })
        .catch(error => {
          console.error(`Error reading ${questionpicking.value}.${questionIdx.value}.txt:`, error);
        });
      }
    }
  });
</script>

<style>
</style>
