<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const STORAGE_KEY = 'fire-calculator-data'

const defaultParams = {
  currentSavings: 1000000,
  currentAge: 30,
  fireAge: 45,
  monthlyExpense: 8000,
  inflationRate: 3,
  returnRate: 5
}

const params = ref({ ...defaultParams })

const savedData = localStorage.getItem(STORAGE_KEY)
if (savedData) {
  try {
    params.value = { ...defaultParams, ...JSON.parse(savedData) }
  } catch (e) {
    console.error('Failed to load saved data:', e)
  }
}

watch(params, (newParams) => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(newParams))
}, { deep: true })

const formatMoney = (value) => {
  if (value >= 100000000) {
    return (value / 100000000).toFixed(2) + '亿'
  } else if (value >= 10000) {
    return (value / 10000).toFixed(2) + '万'
  }
  return value.toLocaleString('zh-CN')
}

const calculateFire = () => {
  const { currentSavings, currentAge, fireAge, monthlyExpense, inflationRate, returnRate } = params.value
  const annualExpense = monthlyExpense * 12
  
  let savings = currentSavings
  const maxYears = 100
  const yearlyData = []
  
  const accumulationYears = Math.max(0, fireAge - currentAge)
  
  for (let i = 0; i < accumulationYears; i++) {
    yearlyData.push({
      year: i + 1,
      age: currentAge + i,
      savings: Math.max(0, savings),
      expense: 0,
      baseExpense: 0,
      inflationImpact: 0,
      phase: 'accumulation'
    })
    savings = savings * (1 + returnRate / 100)
  }
  
  let withdrawalYears = 0
  const startWithdrawalAge = Math.max(currentAge, fireAge)
  
  while (savings > 0 && withdrawalYears < maxYears) {
    const currentExpense = annualExpense * Math.pow(1 + inflationRate / 100, withdrawalYears)
    const inflationImpact = currentExpense - annualExpense
    
    yearlyData.push({
      year: accumulationYears + withdrawalYears + 1,
      age: startWithdrawalAge + withdrawalYears,
      savings: Math.max(0, savings),
      expense: currentExpense,
      baseExpense: annualExpense,
      inflationImpact: inflationImpact,
      phase: 'withdrawal'
    })
    
    savings = savings * (1 + returnRate / 100) - currentExpense
    withdrawalYears++
  }
  
  return {
    totalYears: withdrawalYears,
    endAge: startWithdrawalAge + withdrawalYears - 1,
    yearlyData: yearlyData,
    withdrawalData: yearlyData.filter(d => d.phase === 'withdrawal').slice(0, 20),
    maxSavings: yearlyData.length > 0 ? Math.max(...yearlyData.map(d => d.savings)) : 0,
    chartData: generateChartData(yearlyData)
  }
}

const generateChartData = (yearlyData) => {
  if (yearlyData.length === 0) return []
  
  const accumulationData = yearlyData.filter(d => d.phase === 'accumulation')
  const withdrawalData = yearlyData.filter(d => d.phase === 'withdrawal')
  
  const result = []
  const addedAges = new Set()
  
  const addData = (data) => {
    if (!addedAges.has(data.age)) {
      result.push(data)
      addedAges.add(data.age)
    }
  }
  
  if (accumulationData.length > 0) {
    addData(accumulationData[0])
    if (accumulationData.length > 1) {
      const step = Math.max(1, Math.floor(accumulationData.length / 4))
      for (let i = step; i < accumulationData.length - 1; i += step) {
        addData(accumulationData[i])
      }
      addData(accumulationData[accumulationData.length - 1])
    }
  }
  
  if (withdrawalData.length > 0) {
    addData(withdrawalData[0])
    if (withdrawalData.length > 1) {
      const step = Math.max(1, Math.floor(withdrawalData.length / 10))
      for (let i = step; i < withdrawalData.length - 1; i += step) {
        addData(withdrawalData[i])
      }
      addData(withdrawalData[withdrawalData.length - 1])
    }
  }
  
  return result.sort((a, b) => a.age - b.age)
}

const fireResult = computed(() => {
  return calculateFire()
})

const yearsToFire = computed(() => {
  const { currentAge, fireAge } = params.value
  return Math.max(0, fireAge - currentAge)
})

const resetToDefault = () => {
  params.value = { ...defaultParams }
}
</script>

<template>
  <div class="container">
    <header class="header">
      <h1>FIRE 计算器</h1>
      <p>计算你的财务自由之路，规划提前退休生活</p>
    </header>

    <div class="card">
      <div class="card-header">
        <div class="card-icon">📊</div>
        <h2>基础参数</h2>
      </div>
      <div class="form-grid">
        <div class="form-group">
          <label>当前资金量 (元)</label>
          <input 
            type="number" 
            v-model.number="params.currentSavings" 
            :min="0"
            step="10000"
          />
          <span class="input-hint">当前可用于投资的资产总额</span>
        </div>
        <div class="form-group">
          <label>当前年龄</label>
          <input 
            type="number" 
            v-model.number="params.currentAge" 
            :min="18" 
            :max="100"
          />
        </div>
        <div class="form-group">
          <label>计划FIRE年龄</label>
          <input 
            type="number" 
            v-model.number="params.fireAge" 
            :min="18" 
            :max="100"
          />
          <span class="input-hint">距离FIRE还有 {{ yearsToFire }} 年</span>
        </div>
        <div class="form-group">
          <label>月支出 (元)</label>
          <input 
            type="number" 
            v-model.number="params.monthlyExpense" 
            :min="0"
            step="500"
          />
          <span class="input-hint">FIRE后每月预期支出</span>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <div class="card-icon">⚙️</div>
        <h2>高级参数</h2>
      </div>
      <div class="form-grid">
        <div class="form-group">
          <label>预期通胀率 (%)</label>
          <input 
            type="number" 
            v-model.number="params.inflationRate" 
            :min="0" 
            :max="20"
            step="0.5"
          />
          <span class="input-hint">建议设置 2-4%</span>
        </div>
        <div class="form-group">
          <label>预期年化收益率 (%)</label>
          <input 
            type="number" 
            v-model.number="params.returnRate" 
            :min="0" 
            :max="30"
            step="0.5"
          />
          <span class="input-hint">建议设置 4-8%</span>
        </div>
      </div>
    </div>

    <div class="card result-card">
      <div class="card-header">
        <div class="card-icon">🎯</div>
        <h2>FIRE 计算结果</h2>
      </div>
      
      <div class="result-main">
        <div class="result-years">
          {{ fireResult.totalYears }}<span> 年</span>
        </div>
        <div class="result-age">
          资金可支撑至 {{ fireResult.endAge }} 岁
        </div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <div class="card-icon">📈</div>
        <h2>资产变化趋势</h2>
      </div>
      
      <div class="chart-container">
        <div class="chart-legend">
          <span class="legend-item"><span class="legend-dot accumulation"></span>积累期</span>
          <span class="legend-item"><span class="legend-dot withdrawal"></span>消耗期</span>
        </div>
        <div class="chart">
          <div class="chart-y-axis">
            <span>{{ formatMoney(fireResult.maxSavings) }}</span>
            <span>{{ formatMoney(fireResult.maxSavings / 2) }}</span>
            <span>0</span>
          </div>
          <div class="chart-bars">
            <div 
              v-for="(data, index) in fireResult.chartData" 
              :key="index"
              class="chart-bar-wrapper"
            >
              <div 
                class="chart-bar"
                :class="{
                  'accumulation': data.phase === 'accumulation',
                  'withdrawal': data.phase === 'withdrawal',
                  'danger': data.phase === 'withdrawal' && data.savings < params.monthlyExpense * 12 * 2,
                  'warning': data.phase === 'withdrawal' && data.savings < params.monthlyExpense * 12 * 5 && data.savings >= params.monthlyExpense * 12 * 2
                }"
                :style="{ 
                  height: fireResult.maxSavings > 0 ? Math.max(5, (data.savings / fireResult.maxSavings) * 100) + '%' : '5%'
                }"
                :title="`${data.age}岁: ${formatMoney(data.savings)}${data.phase === 'withdrawal' ? ' (消耗期)' : ' (积累期)'}`"
              ></div>
              <span class="chart-label">{{ data.age }}岁</span>
            </div>
          </div>
        </div>
      </div>
      
      <button class="reset-btn" @click="resetToDefault">
        🔄 重置为默认值
      </button>
    </div>

    <div class="card">
      <div class="card-header">
        <div class="card-icon">📋</div>
        <h2>年度资金明细</h2>
      </div>
      
      <div class="table-container">
        <table class="data-table">
          <thead>
            <tr>
              <th>年份</th>
              <th>年龄</th>
              <th>年初资金</th>
              <th>基础支出</th>
              <th>通胀影响</th>
              <th>实际支出</th>
              <th>年收益</th>
              <th>年末资金</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="(data, index) in fireResult.yearlyData" :key="index">
              <tr v-if="data.phase === 'withdrawal'" :class="{'low-funds': data.savings < params.monthlyExpense * 12 * 3}">
                <td>{{ data.year }}</td>
                <td>{{ data.age }}岁</td>
                <td>{{ formatMoney(data.savings) }}</td>
                <td>{{ formatMoney(data.baseExpense) }}</td>
                <td class="inflation-impact">+{{ formatMoney(data.inflationImpact) }}</td>
                <td>{{ formatMoney(data.expense) }}</td>
                <td class="income">+{{ formatMoney(data.savings * params.returnRate / 100) }}</td>
                <td>{{ formatMoney(Math.max(0, data.savings * (1 + params.returnRate / 100) - data.expense)) }}</td>
              </tr>
            </template>
          </tbody>
        </table>
      </div>
    </div>

    <footer class="footer">
      <p>数据已自动保存至本地浏览器 | 刷新页面不会丢失</p>
    </footer>
  </div>
</template>
