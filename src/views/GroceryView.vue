<script setup lang="ts">
import { computed, ref } from 'vue';
import { RefreshCw, ShoppingBag, CheckCircle2, CheckCircle, Circle, Copy, Image, Users, Minus, Plus } from 'lucide-vue-next';
import { useAppStore } from '@/store';
import { DAYS_ORDER, MEALS_ORDER, DAY_LABELS, MEAL_LABELS } from '@/types';
import { formatQty } from '@/utils/helpers';

const groceryStore = useAppStore();
const { computeGroceryList, toggleGroceryChecked, clearAllGroceryChecked, getServingPeople, setServingPeople } = groceryStore;
const checkedGrocery = computed(() => groceryStore.state.checkedGrocery);
const schedule = computed(() => groceryStore.state.schedule);

const servingPeople = ref(getServingPeople());
const showCustomPeople = ref(false);
const customPeopleValue = ref(servingPeople.value);
const copied = ref(false);
const showImageModal = ref(false);

const updateServingPeople = (delta: number) => {
  const newValue = servingPeople.value + delta;
  if (newValue >= 1 && newValue <= 20) {
    servingPeople.value = newValue;
    setServingPeople(newValue);
  }
};

const setCustomPeople = () => {
  if (customPeopleValue.value >= 1 && customPeopleValue.value <= 20) {
    servingPeople.value = customPeopleValue.value;
    setServingPeople(customPeopleValue.value);
    showCustomPeople.value = false;
  }
};

const groceryKey = (name: string, unit: string) => `${name}|${unit}`;

const items = computed(() => {
  return computeGroceryList().map((i) => ({
    ...i,
    key: groceryKey(i.name, i.unit),
    checked: !!checkedGrocery.value[groceryKey(i.name, i.unit)],
  }));
});

const checkedCount = computed(() => items.value.filter((i) => i.checked).length);
const progressPercent = computed(() =>
  items.value.length === 0 ? 0 : Math.round((checkedCount.value / items.value.length) * 100),
);

const mealPlanSummary = computed(() => {
  const rows: Array<{ day: string; meals: Array<{ label: string; name?: string; emoji?: string }> }> = [];
  for (const d of DAYS_ORDER) {
    const meals = MEALS_ORDER.map((m) => {
      const r = schedule.value[d][m];
      return {
        label: MEAL_LABELS[m],
        name: r?.name,
        emoji: r?.emoji,
      };
    });
    rows.push({ day: DAY_LABELS[d], meals });
  }
  return rows;
});

const showSummary = ref(false);

const confirmReset = () => {
  if (checkedCount.value === 0) return;
  if (confirm('确定重置所有勾选状态吗？')) clearAllGroceryChecked();
};

const toggleAll = () => {
  const allChecked = checkedCount.value === items.value.length && items.value.length > 0;
  for (const it of items.value) {
    if (allChecked && it.checked) {
      toggleGroceryChecked(it.key);
    } else if (!allChecked && !it.checked) {
      toggleGroceryChecked(it.key);
    }
  }
};

const generateText = () => {
  let text = `📋 本周买菜清单（${servingPeople.value}人份）\n\n`;
  text += `━━━━━━━━━━━━\n\n`;
  items.value.forEach((item) => {
    const check = item.checked ? '✓' : '○';
    text += `${check} ${item.name}：${formatQty(item.totalQuantity)}${item.unit}\n`;
  });
  text += `\n━━━━━━━━━━━━\n共 ${items.value.length} 项，已购 ${checkedCount.value} 项`;
  return text;
};

const copyText = async () => {
  const text = generateText();
  try {
    await navigator.clipboard.writeText(text);
    copied.value = true;
    setTimeout(() => {
      copied.value = false;
    }, 2000);
  } catch {
    alert('复制失败，请手动复制');
  }
};

const generateImage = () => {
  showImageModal.value = true;
};

const getImageContent = () => {
  return generateText();
};
</script>

<template>
  <div class="space-y-5">
    <div class="bg-gradient-to-br from-sage-500 to-sage-600 rounded-3xl p-5 shadow-warm text-white relative overflow-hidden">
      <div class="absolute -right-6 -top-6 text-[140px] opacity-10 leading-none select-none">🛒</div>
      <div class="relative z-10">
        <div class="flex items-start justify-between gap-4">
          <div>
            <h3 class="font-display text-xl flex items-center gap-2">
              <ShoppingBag class="w-5 h-5" />
              <span>本周买菜清单</span>
            </h3>
            <p class="text-sm text-white/80 mt-1.5">
              共 <span class="font-bold">{{ items.length }}</span> 项食材
              <span v-if="items.length">，已购 <span class="font-bold">{{ checkedCount }}</span> 项</span>
            </p>
          </div>
          <div class="flex flex-col items-end gap-2">
            <div class="flex items-center gap-2">
              <div class="flex items-center gap-1 bg-white/20 rounded-xl px-3 py-1.5">
                <Users class="w-4 h-4" />
                <button
                  @click="updateServingPeople(-1)"
                  class="w-6 h-6 rounded-lg hover:bg-white/20 flex items-center justify-center transition-colors"
                >
                  <Minus class="w-3 h-3" />
                </button>
                <input
                  v-if="showCustomPeople"
                  v-model.number="customPeopleValue"
                  type="number"
                  min="1"
                  max="20"
                  class="w-12 bg-white/20 rounded-lg text-center text-white font-bold text-sm outline-none"
                  @blur="setCustomPeople"
                  @keyup.enter="setCustomPeople"
                  autofocus
                />
                <button
                  v-else
                  @click="showCustomPeople = true"
                  class="w-12 text-center font-bold"
                >
                  {{ servingPeople }}
                </button>
                <button
                  @click="updateServingPeople(1)"
                  class="w-6 h-6 rounded-lg hover:bg-white/20 flex items-center justify-center transition-colors"
                >
                  <Plus class="w-3 h-3" />
                </button>
                <span class="text-xs text-white/70">人份</span>
              </div>
            </div>
            <div class="flex gap-2">
              <button
                @click="copyText"
                class="h-9 px-3.5 rounded-xl bg-white/20 hover:bg-white/30 backdrop-blur text-white text-xs font-medium transition-all flex items-center gap-1"
              >
                <Copy class="w-3.5 h-3.5" />
                <span>{{ copied ? '已复制' : '复制文本' }}</span>
              </button>
              <button
                @click="generateImage"
                class="h-9 px-3.5 rounded-xl bg-white/20 hover:bg-white/30 backdrop-blur text-white text-xs font-medium transition-all flex items-center gap-1"
              >
                <Image class="w-3.5 h-3.5" />
                <span>生成图片</span>
              </button>
            </div>
          </div>
        </div>

        <div v-if="items.length > 0" class="mt-4">
          <div class="flex items-center justify-between text-xs text-white/80 mb-1.5">
            <span>采购进度</span>
            <span>{{ progressPercent }}%</span>
          </div>
          <div class="h-2.5 bg-white/20 rounded-full overflow-hidden backdrop-blur">
            <div
              class="h-full bg-white rounded-full transition-all duration-500"
              :style="{ width: `${progressPercent}%` }"
            ></div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showSummary" class="bg-white rounded-3xl p-5 shadow-card border border-warm-100 animate-fade-in">
      <h4 class="font-display text-lg text-clay-800 mb-3 flex items-center gap-2">
        <span>📋</span>
        <span>本周菜单摘要</span>
      </h4>
      <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-7 gap-3">
        <div
          v-for="row in mealPlanSummary"
          :key="row.day"
          class="rounded-2xl border border-warm-100 bg-warm-50/50 p-3"
        >
          <div class="font-display text-warm-600 text-sm mb-2 text-center">{{ row.day }}</div>
          <div class="space-y-1.5">
            <div v-for="m in row.meals" :key="m.label" class="text-xs">
              <span class="text-clay-500">{{ m.label }}：</span>
              <span v-if="m.name" class="text-clay-700 font-medium">
                <span>{{ m.emoji }}</span> {{ m.name }}
              </span>
              <span v-else class="text-clay-300">—</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="items.length === 0" class="bg-white rounded-3xl p-12 shadow-card border border-warm-100 text-center">
      <div class="text-6xl mb-4">🥗</div>
      <div class="font-display text-xl text-clay-800 mb-2">还没有安排菜呢</div>
      <div class="text-sm text-clay-500">先去「本周排餐」选几道菜，食材清单会自动生成哦~</div>
    </div>

    <div v-else class="space-y-4">
      <div class="flex items-center justify-between">
        <div class="text-sm text-clay-500">
          点击卡片或勾选框，标记已买到的食材 🍀
        </div>
        <div class="flex items-center gap-2">
          <button
            @click="showSummary = !showSummary"
            class="h-9 px-3.5 rounded-xl bg-white border border-warm-100 text-sm font-medium text-clay-600 hover:bg-warm-50 transition-all"
          >
            {{ showSummary ? '收起菜单' : '查看菜单' }}
          </button>
          <button
            @click="toggleAll"
            class="h-9 px-3.5 rounded-xl bg-white border border-warm-100 text-sm font-medium text-clay-600 hover:bg-warm-50 transition-all flex items-center gap-1.5"
          >
            <component
              :is="checkedCount === items.length && items.length > 0 ? CheckCircle : Circle"
              class="w-4 h-4"
            />
            <span>{{ checkedCount === items.length && items.length > 0 ? '全部取消' : '全部勾选' }}</span>
          </button>
        </div>
      </div>

      <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-3">
        <div
          v-for="item in items"
          :key="item.key"
          @click="toggleGroceryChecked(item.key)"
          class="rounded-2xl p-4 border-2 bg-white cursor-pointer transition-all hover:shadow-card group select-none"
          :class="item.checked
            ? 'border-sage-200 bg-sage-50/50'
            : 'border-warm-100 hover:border-warm-300'"
        >
          <div class="flex items-start gap-3">
            <input
              type="checkbox"
              class="custom-checkbox mt-0.5"
              :checked="item.checked"
              @click.stop="toggleGroceryChecked(item.key)"
            />
            <div class="flex-1 min-w-0">
              <div class="flex items-baseline justify-between gap-2">
                <div
                  class="font-medium text-base"
                  :class="item.checked ? 'text-clay-400 line-through' : 'text-clay-800'"
                >
                  {{ item.name }}
                </div>
                <div
                  class="font-display text-lg flex-shrink-0 whitespace-nowrap"
                  :class="item.checked ? 'text-sage-400' : 'text-warm-600'"
                >
                  {{ formatQty(item.totalQuantity) }}
                  <span class="text-sm font-sans opacity-80 ml-0.5">{{ item.unit }}</span>
                </div>
              </div>
              <div
                v-if="item.usedIn.length"
                class="text-xs mt-1.5 leading-relaxed"
                :class="item.checked ? 'text-clay-400' : 'text-clay-500'"
              >
                用于：{{ item.usedIn.join('、') }}
              </div>
            </div>
            <CheckCircle2
              v-if="item.checked"
              class="w-5 h-5 text-sage-500 flex-shrink-0 animate-bounce-soft"
            />
          </div>
        </div>
      </div>

      <div
        v-if="checkedCount === items.length && items.length > 0"
        class="bg-gradient-to-r from-sage-500 to-sage-600 rounded-3xl p-6 text-white text-center shadow-warm animate-slide-up"
      >
        <div class="text-5xl mb-2">🎉</div>
        <div class="font-display text-2xl">所有食材都买齐啦！</div>
        <div class="text-sm text-white/80 mt-1">准备好做一顿丰盛的家常菜吧~</div>
      </div>
    </div>

    <Teleport to="body">
      <div v-if="showImageModal" class="fixed inset-0 z-50 flex items-center justify-center p-4 animate-fade-in" @click.self="showImageModal = false">
        <div class="absolute inset-0 bg-clay-900/60 backdrop-blur-sm"></div>
        <div class="relative bg-white rounded-3xl shadow-warm max-w-md w-full max-h-[90vh] overflow-hidden animate-slide-up">
          <div class="px-6 py-4 border-b border-warm-100 flex items-center justify-between">
            <h3 class="font-display text-lg text-clay-800">📸 买菜清单图片</h3>
            <button
              @click="showImageModal = false"
              class="w-8 h-8 rounded-lg hover:bg-warm-100 flex items-center justify-center text-clay-400"
            >
              ✕
            </button>
          </div>
          <div class="p-6">
            <div class="bg-gradient-to-br from-sage-500 to-sage-600 rounded-2xl p-6 text-white">
              <div class="text-4xl text-center mb-4">🛒</div>
              <div class="font-display text-xl text-center mb-4">本周买菜清单</div>
              <div class="text-sm text-white/80 text-center mb-4">{{ servingPeople }}人份</div>
              <div class="border-t border-white/20 pt-4">
                <pre class="text-sm whitespace-pre-wrap text-left font-sans">{{ getImageContent().replace(/📋|━━━━━━━━━━━━|\n\n/g, '\n').trim() }}</pre>
              </div>
            </div>
          </div>
          <div class="px-6 pb-6">
            <button
              @click="copyText(); showImageModal = false"
              class="w-full h-12 rounded-xl bg-warm-500 text-white font-medium hover:bg-warm-600 transition-colors flex items-center justify-center gap-2"
            >
              <Copy class="w-4 h-4" />
              <span>复制清单文本</span>
            </button>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>