<script setup lang="ts">
import { computed, ref, watch } from 'vue';
import { useDictionaryStore } from '~/store/dictionary';
import type { ReviewBatch, ReviewBatchItem } from '~/types/dictionary';

const visible = defineModel<boolean>({ required: true });
const store = useDictionaryStore();

const expandedId = ref('');
const createOpen = ref(false);
const draftTitle = ref('');
const draftReviewer = ref('主审·和老师');
const draftQuery = ref('');
const pickedIds = ref<string[]>([]);

const statusMeta = {
  draft: { label: '草稿', theme: 'default' },
  review: { label: '待审', theme: 'warning' },
  disputed: { label: '争议', theme: 'danger' },
  confirmed: { label: '已确认', theme: 'success' }
} as const;

const itemMeta = {
  pending: { label: '待审', theme: 'warning' },
  approved: { label: '已通过', theme: 'success' },
  returned: { label: '已退回', theme: 'danger' }
} as const;

const sortedBatches = computed(() => [...store.batches].sort((a, b) => a.status === b.status
  ? b.createdAt.localeCompare(a.createdAt)
  : a.status === 'open' ? -1 : 1));
const closedCount = computed(() => store.batches.filter((batch) => batch.status === 'closed').length);

const stats = (batch: ReviewBatch) => {
  const approved = batch.items.filter((item) => item.status === 'approved').length;
  const returned = batch.items.filter((item) => item.status === 'returned').length;
  return { total: batch.items.length, approved, returned, pending: batch.items.length - approved - returned };
};

const entryOf = (item: ReviewBatchItem) => store.entries.find((entry) => entry.id === item.entryId);

const pickableEntries = computed(() => {
  const term = draftQuery.value.trim().toLowerCase();
  const order = { review: 0, disputed: 1, draft: 2, confirmed: 3 } as const;
  return [...store.entries]
    .filter((entry) => !term || [entry.headword, entry.definition, entry.pronunciation, entry.partOfSpeech].join(' ').toLowerCase().includes(term))
    .sort((a, b) => order[a.status] - order[b.status]);
});

watch(visible, (open) => {
  if (open) expandedId.value = store.openBatches[0]?.id ?? store.batches[0]?.id ?? '';
});

watch(createOpen, (open) => {
  if (!open) return;
  draftTitle.value = `审校批次 ${new Date().toLocaleDateString('zh-CN')}`;
  draftReviewer.value = '主审·和老师';
  draftQuery.value = '';
  pickedIds.value = [];
});

const pickAll = () => {
  pickedIds.value = pickableEntries.value.filter((entry) => !store.openBatchOf.has(entry.id)).map((entry) => entry.id);
};

const submitCreate = () => {
  const batch = store.createBatch(draftTitle.value, draftReviewer.value, pickedIds.value);
  if (!batch) return;
  createOpen.value = false;
  expandedId.value = batch.id;
};

const viewEntry = (entryId: string) => {
  store.selectedId = entryId;
  visible.value = false;
};
</script>

<template>
  <t-drawer v-model:visible="visible" header="审校批次" size="660px" :footer="false">
    <div class="batch-drawer">
      <div class="batch-intro">
        <div class="batch-intro-counts">
          <div><strong>{{ store.openBatches.length }}</strong><span>进行中批次</span></div>
          <div><strong>{{ closedCount }}</strong><span>已结束</span></div>
        </div>
        <p>主审从词条库挑选词条发起批次，逐条通过或退回；全部通过后才能结束。通过后内容再变更的条目会自动标回待审，同一词条不会进入两个未结束的批次。</p>
        <t-button block theme="primary" size="small" @click="createOpen = true">＋ 发起审校批次</t-button>
      </div>

      <div class="batch-list">
        <article v-for="batch in sortedBatches" :key="batch.id" class="batch-card" :class="{ closed: batch.status === 'closed' }">
          <header @click="expandedId = expandedId === batch.id ? '' : batch.id">
            <div class="batch-head-top">
              <t-tag size="small" variant="light" :theme="batch.status === 'open' ? 'warning' : 'success'">{{ batch.status === 'open' ? '进行中' : '已结束' }}</t-tag>
              <strong>{{ batch.title }}</strong>
              <span class="batch-toggle">{{ expandedId === batch.id ? '收起 ▲' : '展开 ▼' }}</span>
            </div>
            <div class="batch-head-sub">
              <span>{{ batch.reviewer }}</span>
              <span>发起于 {{ new Date(batch.createdAt).toLocaleString('zh-CN') }}</span>
              <span v-if="batch.closedAt">结束于 {{ new Date(batch.closedAt).toLocaleString('zh-CN') }}</span>
              <span>通过 {{ stats(batch).approved }} · 退回 {{ stats(batch).returned }} · 待审 {{ stats(batch).pending }} / 共 {{ stats(batch).total }} 条</span>
            </div>
            <div class="batch-bar"><i :style="{ width: `${stats(batch).total ? Math.round((stats(batch).approved / stats(batch).total) * 100) : 0}%` }" /></div>
          </header>

          <div v-if="expandedId === batch.id" class="batch-body">
            <div v-for="(item, index) in batch.items" :key="item.entryId" class="batch-item">
              <span class="item-index">{{ String(index + 1).padStart(2, '0') }}</span>
              <div class="item-main">
                <template v-if="entryOf(item)">
                  <strong>{{ entryOf(item)!.headword || '未命名词条' }}</strong>
                  <small>[{{ entryOf(item)!.pronunciation || '音标待补' }}] · {{ entryOf(item)!.partOfSpeech || '词性待定' }}</small>
                </template>
                <template v-else>
                  <strong>词条已删除</strong>
                  <small>该词条已从词条库移除</small>
                </template>
                <small v-if="item.status === 'pending' && item.reopenedAt" class="item-hint">内容在审校后发生变化，{{ new Date(item.reopenedAt).toLocaleString('zh-CN') }} 标回待审</small>
                <small v-else-if="item.decidedAt">{{ itemMeta[item.status].label }}于 {{ new Date(item.decidedAt).toLocaleString('zh-CN') }}</small>
              </div>
              <t-tag v-if="entryOf(item)" size="small" variant="light" :theme="statusMeta[entryOf(item)!.status].theme">{{ statusMeta[entryOf(item)!.status].label }}</t-tag>
              <t-tag size="small" variant="light" :theme="itemMeta[item.status].theme">{{ item.status === 'pending' && item.reopenedAt ? '待重审' : itemMeta[item.status].label }}</t-tag>
              <div v-if="batch.status === 'open' && entryOf(item)" class="item-actions">
                <t-button size="small" variant="text" @click="viewEntry(item.entryId)">查看</t-button>
                <t-button size="small" theme="success" variant="outline" :disabled="item.status === 'approved'" @click="store.decideBatchItem(batch.id, item.entryId, 'approved')">通过</t-button>
                <t-button size="small" theme="danger" variant="outline" :disabled="item.status === 'returned'" @click="store.decideBatchItem(batch.id, item.entryId, 'returned')">退回</t-button>
              </div>
            </div>
            <t-empty v-if="!batch.items.length" description="批次成员已被全部移除" />
            <footer v-if="batch.status === 'open'" class="batch-foot">
              <span v-if="stats(batch).pending + stats(batch).returned > 0">还有 {{ stats(batch).pending + stats(batch).returned }} 条未通过，全部通过后才能结束批次</span>
              <span v-else>全部条目已通过，可以结束批次</span>
              <t-popconfirm content="结束后批次不可再修改，确认结束？" @confirm="store.closeBatch(batch.id)">
                <t-button size="small" theme="primary" :disabled="stats(batch).pending + stats(batch).returned > 0">结束批次</t-button>
              </t-popconfirm>
            </footer>
            <footer v-else class="batch-foot"><span>批次已结束，审校结果已归档，可在版本记录中追溯</span></footer>
          </div>
        </article>
        <t-empty v-if="!store.batches.length" description="还没有审校批次，点击上方按钮发起" />
      </div>
    </div>

    <t-dialog v-model:visible="createOpen" header="发起审校批次" width="680px" :footer="false">
      <div class="batch-create">
        <div class="field-grid two">
          <label class="field-block"><span>批次名称</span><t-input v-model="draftTitle" placeholder="例如：九月田野词条审校" /></label>
          <label class="field-block"><span>主审</span><t-input v-model="draftReviewer" placeholder="主审人" /></label>
        </div>
        <div class="pick-tools">
          <t-input v-model="draftQuery" clearable placeholder="搜索词形、释义、发音…" />
          <t-button size="small" variant="outline" @click="pickAll">全选可选</t-button>
          <t-button size="small" variant="outline" :disabled="!pickedIds.length" @click="pickedIds = []">清空</t-button>
        </div>
        <div class="pick-list">
          <label v-for="entry in pickableEntries" :key="entry.id" class="pick-row" :class="{ disabled: store.openBatchOf.has(entry.id) }">
            <input v-model="pickedIds" type="checkbox" :value="entry.id" :disabled="store.openBatchOf.has(entry.id)" />
            <strong>{{ entry.headword || '未命名词条' }}</strong>
            <t-tag size="small" variant="light" :theme="statusMeta[entry.status].theme">{{ statusMeta[entry.status].label }}</t-tag>
            <span class="pick-def">{{ entry.definition || '尚未填写释义' }}</span>
            <span v-if="store.openBatchOf.has(entry.id)" class="busy-tag">已在《{{ store.openBatchOf.get(entry.id) }}》</span>
          </label>
          <t-empty v-if="!pickableEntries.length" description="没有符合条件的词条" />
        </div>
        <p class="pick-hint">同一词条不能同时出现在两个未结束的批次中；已在其他批次中的词条不可勾选，批次结束后可再次发起。</p>
        <div class="dialog-actions">
          <span class="pick-count">已选 {{ pickedIds.length }} 条</span>
          <t-button variant="outline" @click="createOpen = false">取消</t-button>
          <t-button theme="primary" :disabled="!pickedIds.length" @click="submitCreate">发起批次</t-button>
        </div>
      </div>
    </t-dialog>
  </t-drawer>
</template>
