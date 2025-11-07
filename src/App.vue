<template>
  <div class="app-container">
    <header class="app-header">
        <div class="header-stats">
          <h1>活动计算器 - 显示 {{ displayMonthsCount }} 个月</h1>
          <div class="month-controls">
            <el-button @click="navigateMonths(-1)" type="primary" plain>
              <span>← 上移</span>
            </el-button>
            <el-button @click="currentDate = new Date()" type="success" plain>当前月份</el-button>
            <el-button @click="navigateMonths(1)" type="primary" plain>
              <span>下移 →</span>
            </el-button>
            <el-button type="primary" @click="dialogVisible = true" class="settings-button">
              设置
            </el-button>
          </div>
        </div>
      </header>
    
      <main class="app-main">
        <div class="year-calendar">
          <div class="month-container" v-for="(month, index) in displayMonths" :key="index">
            <h3 class="month-title">{{ month.getFullYear() }}年{{ month.getMonth() + 1 }}月</h3>
            <el-calendar :range="[month, new Date(month.getFullYear(), month.getMonth() + 1, 0)]">
              <template #date-cell="{ data }">
                <div 
                  class="date-cell" 
                  :class="{ 
                    'checked-date': new Date(data.day).getMonth() === month.getMonth() &&checkedDates.has(data.day),
                    'current-month': data.type === 'current-month'
                  }" 
                  @click="handleDateCellClick(data, month)"
                >
                  <template v-if="new Date(data.day).getMonth() === month.getMonth()">
                    <span class="date-number">{{ new Date(data.day).getDate() }}</span>
                    <el-icon v-if="checkedDates.has(data.day)" class="check-icon"><Check /></el-icon>
                  </template>
                </div>
              </template>
            </el-calendar>
          </div>
        </div>
      </main>
    
    <footer class="app-footer">
      <p>点击日期进行标记/取消标记</p>
    </footer>
    
    <!-- 设置对话框 -->
    <el-dialog v-model="dialogVisible" title="标记设置" width="50%" destroy-on-close>
      <div class="dialog-content">
        <div class="setting-item">
          <label class="setting-label">显示月数：</label>
          <el-input-number
            v-model="displayMonthsCount"
            :min="1"
            :max="12"
            :step="1"
            class="setting-input"
          />
          <small class="setting-hint">（默认显示3个月，以当前月份为中心）</small>
        </div>
        <div class="setting-item">
          <label class="setting-label">检查窗口天数：</label>
          <el-input-number
            v-model="daysWindow"
            :min="1"
            :max="365"
            :step="1"
            class="setting-input"
          />
        </div>
        <div class="setting-item">
          <label class="setting-label">允许的最大标记数量：</label>
          <el-input-number
            v-model="threshold"
            :min="1"
            :max="daysWindow"
            :step="1"
            class="setting-input"
          />
        </div>
      </div>
      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" @click="saveThreshold">保存</el-button>
      </template>
    </el-dialog>
  </div>
</template>
<script setup>
import { ref, onMounted, computed } from 'vue';
import { ElCalendar, ElDialog, ElInputNumber, ElButton, ElMessage, ElMessageBox } from 'element-plus';
import { Check } from '@element-plus/icons-vue';

// 存储勾选的日期
const checkedDates = ref(new Set());
// 设置阈值的对话框状态
const dialogVisible = ref(false);
// 阈值设置
const threshold = ref(63); // 默认阈值为63
// 天数窗口设置
const daysWindow = ref(90); // 默认90天窗口
// 当前日期
const currentDate = ref(new Date());
// 显示的月数设置，默认3个月
const displayMonthsCount = ref(3);

// 根据设置的月数和当前日期生成要显示的月份
const displayMonths = computed(() => {
  const months = [];
  const startMonth = new Date(currentDate.value.getFullYear(), currentDate.value.getMonth() - Math.floor((displayMonthsCount.value - 1) / 2), 1);
  
  for (let i = 0; i < displayMonthsCount.value; i++) {
    const month = new Date(startMonth.getFullYear(), startMonth.getMonth() + i, 1);
    months.push(month);
  }
  
  return months;
});

// 不需要多月份显示，ElCalendar组件自带月份切换功能

// 从localStorage加载数据
const loadCheckedDates = () => {
  const saved = localStorage.getItem('checkedDates');
  const savedThreshold = localStorage.getItem('threshold');
  const savedDaysWindow = localStorage.getItem('daysWindow');
  const savedDisplayMonthsCount = localStorage.getItem('displayMonthsCount');
  if (saved) {
    checkedDates.value = new Set(JSON.parse(saved));
  }
  if (savedThreshold) {
    threshold.value = parseInt(savedThreshold);
  }
  if (savedDaysWindow) {
    daysWindow.value = parseInt(savedDaysWindow);
  }
  if (savedDisplayMonthsCount) {
    displayMonthsCount.value = parseInt(savedDisplayMonthsCount);
  }
};

// 保存数据到localStorage
const saveData = () => {
  localStorage.setItem('checkedDates', JSON.stringify([...checkedDates.value]));
  localStorage.setItem('threshold', threshold.value.toString());
  localStorage.setItem('daysWindow', daysWindow.value.toString());
  localStorage.setItem('displayMonthsCount', displayMonthsCount.value.toString());
};

// 处理日期单元格点击
  const handleDateCellClick = (data, currentMonth) => {
    // 修复当月第一行日期无法选中的问题
    // 不要仅通过type判断，而是通过日期的月份与当前月份比较
    const date = new Date(data.day);
    const dateMonth = date.getMonth();
    const currentMonthNum = currentMonth.getMonth();
    
    // 只有当月的日期才能被选中
    if (dateMonth !== currentMonthNum) {
      return;
    }

    const dateStr = date.toISOString().split('T')[0]; // 格式化为YYYY-MM-DD
    
    // 检查如果添加这个日期是否会导致超过阈值
    let willExceedThreshold = false;
    let nstartDate = null;
    let nendDate = null;

    if (!checkedDates.value.has(dateStr)) {
      // 临时添加日期进行预检查
      checkedDates.value.add(dateStr);
      
      // 使用临时数组进行检查，避免实际修改数据
      const tempDates = [...checkedDates.value].sort();
      
      // 检查包含该日期的所有可能窗口
      for (let i = 0; i < tempDates.length; i++) {
        if(willExceedThreshold) break;
        const startDate = new Date(tempDates[i]);
        const endDate = new Date(startDate);
        endDate.setDate(endDate.getDate() + daysWindow.value - 1);
        
        // 只检查包含当前日期的窗口
        if (startDate > date || endDate < date) continue;
        
        let count = 0;
        for (let j = i; j < tempDates.length; j++) {
          const currentDate = new Date(tempDates[j]);
          if (currentDate <= endDate) {
            count++;
          } else {
            break;
          }
        }
        
        if (count > threshold.value) {
          nstartDate = startDate;
          nendDate = endDate;
          willExceedThreshold = true;
          break;
        }
      }
      
      // 移除临时添加的日期
      checkedDates.value.delete(dateStr);
      
      // 如果会超过阈值，显示警告并阻止添加
      if (willExceedThreshold) {
        ElMessageBox.confirm(
          `添加此日期会导致在${daysWindow.value}天窗口内超过设置的最大阈值${threshold.value}！\n` +
          `当前窗口为：${formatDate(nstartDate)} 至 ${formatDate(nendDate)}`,
          '标记超限提醒',
          {
            confirmButtonText: '确认',
            cancelButtonText: '取消',
            type: 'warning'
          }
        );
        // return;
      }
    }
    
    // 如果通过了阈值检查，正常添加或移除日期
    if (checkedDates.value.has(dateStr)) {
      checkedDates.value.delete(dateStr);
    } else {
      checkedDates.value.add(dateStr);
    }
    saveData();
    // 检查窗口内是否超过阈值（虽然预检查已做，但为了确保完整性）
    // checkThresholdViolation(date);
  };

// 检查指定天数窗口内是否超过阈值
const checkThresholdViolation = (specificDate = null) => {
  let willExceedThreshold = false;
  const dates = [...checkedDates.value].sort();
  // 如果提供了特定日期（即刚刚勾选的日期），只检查包含该日期的窗口
  if (specificDate) {
    const checkDate = new Date(specificDate);
    
    // 计算包含该日期的所有可能窗口
    // 窗口的起始日期可以是从checkDate减去(daysWindow-1)天到checkDate
    const windowStart = new Date(checkDate);
    windowStart.setDate(windowStart.getDate() - (daysWindow.value - 1));
    
    // 检查从windowStart到checkDate的每个可能的窗口起始点
    // 但为了效率，我们只需要检查实际有标记的日期作为窗口起始点
    for (let i = 0; i < dates.length; i++) {
      const startDate = new Date(dates[i]);
      
      // 只检查可能包含specificDate的窗口
      if (startDate > checkDate) continue;
      
      const endDate = new Date(startDate);
      endDate.setDate(endDate.getDate() + daysWindow.value - 1); // 窗口包含开始日期
      
      // 如果窗口不包含specificDate，跳过
      if (endDate < checkDate) continue;
      
      let count = 0;
      
      for (let j = i; j < dates.length; j++) {
        const currentDate = new Date(dates[j]);
        if (currentDate <= endDate) {
          count++;
        } else {
          break;
        }
      }
      if (count > threshold.value) {
        willExceedThreshold = true;
      }
    }
  }
  return willExceedThreshold;
};

// 格式化日期显示
const formatDate = (date) => {
  return date.toLocaleDateString('zh-CN', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });
};

// 保存设置
const saveThreshold = () => {
  saveData();
  dialogVisible.value = false;
  ElMessage.success('设置已保存');
};

// 月份导航函数
const navigateMonths = (direction) => {
  // 根据方向移动月份视图
  currentDate.value = new Date(currentDate.value.getFullYear(), currentDate.value.getMonth() + direction, 1);
};

// 生命周期钩子
onMounted(() => {
  loadCheckedDates();
});
</script>

<style>
/* 全局样式重置 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background-color: #f5f7fa;
}

.app-container {
  width: 100%;
  min-height: 100vh;
  background-color: #fff;
  display: flex;
  flex-direction: column;
}

.app-header {
  padding: 20px;
  background: linear-gradient(135deg, #409eff 0%, #66b1ff 100%);
  color: white;
}

.header-stats {
  display: flex;
  justify-content: space-between;
  align-items: center;
  max-width: 1400px;
  margin: 0 auto;
  gap: 20px;
  flex-wrap: wrap;
}

.header-stats h1 {
  font-size: 28px;
  font-weight: 600;
  margin: 0;
}

.month-controls {
  display: flex;
  gap: 10px;
}

.setting-hint {
  color: #909399;
  font-size: 12px;
  margin-top: 4px;
  display: block;
}

.settings-button {
  font-size: 14px;
  padding: 8px 20px;
  white-space: nowrap;
  min-width: 80px;
}

.app-main {
  flex: 1;
  padding: 20px;
}

/* 一年日历布局 - 电脑端优化 */
.year-calendar {
    max-width: 1400px;
    margin: 0 auto;
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
    gap: 20px;
    justify-items: stretch;
  }
  
  .month-container {
    width: 100%;
  }

.month-container {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 2px 16px rgba(0, 0, 0, 0.08);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.month-container:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 24px rgba(0, 0, 0, 0.12);
}

.month-title {
  text-align: center;
  font-size: 18px;
  font-weight: 600;
  margin-bottom: 16px;
  color: #303133;
  padding-bottom: 10px;
  border-bottom: 2px solid #ecf5ff;
}

/* 日历样式调整 */
.el-calendar {
  width: 100%;
  border: none;
  box-shadow: none;
}

.el-calendar__header {
  display: none; /* 隐藏默认的月份标题，使用我们自定义的 */
}

.el-calendar__body {
  padding: 0;
}

/* 自定义日期单元格 */
.date-cell {
  position: relative;
  height: 52px;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.3s ease;
  border-radius: 8px;
}

.date-cell:hover {
  background-color: #ecf5ff;
  transform: scale(1.05);
}

.date-number {
  font-size: 16px;
  font-weight: 500;
}

.check-icon {
  position: absolute;
  bottom: 4px;
  right: 4px;
  color: #409eff;
  font-size: 14px;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: scale(0.5); }
  to { opacity: 1; transform: scale(1); }
}

.checked-date {
  background: linear-gradient(135deg, #409eff 0%, #66b1ff 100%);
  color: white !important;
  border-radius: 8px;
  font-weight: bold;
  animation: pulse 0.3s ease;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.1); }
  100% { transform: scale(1); }
}

.checked-date .date-number {
  color: white;
  font-weight: 600;
}

.checked-date .check-icon {
  color: white;
}

/* 隐藏上个月和下个月的日期 */
.el-calendar-table:not(.is-range) td.prev,
.el-calendar-table:not(.is-range) td.next {
  display: none;
}

/* 确保当前月的日期正确显示 */
.current-month .date-number {
  visibility: visible !important;
}

/* 禁用超过阈值的日期 */
.date-disabled {
  background-color: #f5f7fa;
  color: #c0c4cc;
  cursor: not-allowed;
}

.app-footer {
  padding: 16px;
  text-align: center;
  background-color: #f5f7fa;
  color: #606266;
  font-size: 14px;
  border-top: 1px solid #ebeef5;
}

/* 对话框样式 */
.dialog-content {
  padding: 16px 0;
}

.setting-item {
  margin-bottom: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.setting-label {
  font-weight: 500;
  color: #606266;
  font-size: 14px;
}

.setting-input {
  width: 100%;
}

/* 弹性布局响应式优化 */
@media (min-width: 900px) {
  .year-calendar {
    grid-template-columns: repeat(3, 1fr); /* 大屏幕最多3列 */
    gap: 25px;
  }
}

@media (min-width: 600px) and (max-width: 899px) {
  .year-calendar {
    grid-template-columns: repeat(2, 1fr); /* 中等屏幕2列 */
    gap: 20px;
  }
}

/* 移动端响应式调整 */
@media (max-width: 599px) {
  .year-calendar {
    grid-template-columns: 1fr; /* 移动端显示1列 */
    gap: 15px;
  }
  
  .header-stats {
    flex-direction: column;
    align-items: flex-start;
    gap: 15px;
  }
  
  .month-controls {
    flex-wrap: wrap;
    gap: 8px;
  }
  
  .month-controls .el-button {
    flex: 1;
    min-width: 0;
  }
  
  .settings-button {
    width: 100%;
  }
  
  .app-main {
    padding: 16px;
  }
  
  .date-number {
    font-size: 14px;
  }
  
  .date-cell {
    height: 44px;
  }
}
</style>
