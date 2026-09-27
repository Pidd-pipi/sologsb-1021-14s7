<script setup lang="ts">
import { computed, ref } from 'vue';
import type { EntryStatus, ReviewBatch } from '~/types/dictionary';
import { useDictionaryStore } from '~/store/dictionary';

const visible = defineModel<boolean>({ required: true });
const store = useDictionaryStore();

const creating = ref(false);
const draftTitle = ref('');
const pickerQuery = ref('');
const picked = ref<string[]>([]);
const returnFor = ref('');
const returnNote = ref('');

const statusLabels: Record<EntryStatus, string> = { draft: '草稿', review: '待审', disputed: '争议', confirmed: '已确认' };
const itemMeta = {
  pending: { label: '待审', theme: 'warning' },
  approved: { label: '已通过', theme: 'success' },
  returned: { label: '已退回', theme: 'danger' }
} as const;

const entryOf = (entryId: string) => store.entries.find((entry) => entry.id === entryId);
const busyBatchOf = (entryId: string) => store.openBatches.find((batch) => batch.items.some((item) => item.entryId === entryId));
const counts = (batch: ReviewBatch) => ({
  approved: batch.items.filter((item) => item.status === 'approved').length,
  returned: batch.items.filter((item) => item.status === 'returned').length,
  pending: batch.items.filter((item) => item.status === 'pending').length
});

const pickerEntries = computed(() => {
  const term = pickerQuery.value.trim().toLowerCase();
  if (!term) return store.entries;
  return store.entries.filter((entry) => [entry.headword, entry.definition, entry.pronunciation, ...entry.synonyms].join(' ').toLowerCase().includes(term));
});

const togglePick = (entryId: string) => {
  if (busyBatchOf(entryId)) return;
  picked.value = picked.value.includes(entryId) ? picked.value.filter((id) => id !== entryId) : [...picked.value, entryId];
};
const pickAll = () => {
  picked.value = [...new Set([...picked.value, ...pickerEntries.value.filter((entry) => !busyBatchOf(entry.id)).map((entry) => entry.id)])];
};
const cancelCreate = () => {
  creating.value = false;
  draftTitle.value = '';
  pickerQuery.value = '';
  picked.value = [];
};
const submitCreate = () => {
  if (store.createBatch(draftTitle.value, picked.value)) cancelCreate();
};

const openReturn = (entryId: string) => {
  returnFor.value = returnFor.value === entryId ? '' : entryId;
  returnNote.value = '';
};
const submitReturn = (batchId: string, entryId: string) => {
  store.decideBatchItem(batchId, entryId, 'returned', returnNote.value);
  returnFor.value = '';
  returnNote.value = '';
};
const jumpTo = (entryId: string) => {
  store.selectedId = entryId;
  visible.value = false;
};
</script>

<template>
  <t-drawer v-model:visible="visible" header="审校批次" size="640px" :footer="false">
    <div class="batch-drawer">
      <div class="batch-intro">
        <div>
          <strong>{{ store.openBatches.length }}</strong><span>个进行中批次</span>
          <p>主审从词条库挑选词条发起批次，逐条通过或退回；全部通过后批次才能结束，数据保存在浏览器本地。</p>
        </div>
        <t-button v-if="!creating" size="small" theme="primary" @click="creating = true">＋ 发起批次</t-button>
      </div>

      <div v-if="creating" class="batch-create">
        <div class="create-row">
          <t-input v-model="draftTitle" placeholder="批次名称，如：2026-09 田野词条审校" />
          <t-input v-model="pickerQuery" clearable placeholder="搜索词形、释义、同义词…" />
        </div>
        <div class="picker-toolbar">
          <span>已选 <strong>{{ picked.length }}</strong> 条</span>
          <button class="text-action" @click="pickAll">全选可入批词条</button>
          <button class="text-action" @click="picked = []">清空</button>
        </div>
        <div class="picker-list">
          <div
            v-for="entry in pickerEntries"
            :key="entry.id"
            class="picker-row"
            :class="{ disabled: !!busyBatchOf(entry.id), on: picked.includes(entry.id) }"
            @click="togglePick(entry.id)"
          >
            <span class="picker-check">{{ picked.includes(entry.id) ? '✓' : '' }}</span>
            <div class="picker-main">
              <strong>{{ entry.headword || '未命名词条' }}</strong>
              <span>[{{ entry.pronunciation || '音标待补' }}] · {{ statusLabels[entry.status] }}</span>
              <p>{{ entry.definition || '尚未填写释义' }}</p>
            </div>
            <span v-if="busyBatchOf(entry.id)" class="picker-busy">已在「{{ busyBatchOf(entry.id)?.title }}」</span>
          </div>
          <t-empty v-if="!pickerEntries.length" description="没有符合条件的词条" />
        </div>
        <div class="dialog-actions">
          <t-button variant="outline" @click="cancelCreate">取消</t-button>
          <t-button theme="primary" :disabled="!picked.length" @click="submitCreate">发起批次（{{ picked.length }} 条）</t-button>
        </div>
      </div>

      <div class="batch-list">
        <article v-for="batch in store.openBatches" :key="batch.id" class="batch-card">
          <header class="batch-card-head">
            <div>
              <div class="batch-title"><strong>{{ batch.title }}</strong><t-tag size="small" theme="warning" variant="light">进行中</t-tag></div>
              <span class="batch-meta">{{ batch.reviewer }} · 发起于 {{ new Date(batch.createdAt).toLocaleString('zh-CN') }}</span>
            </div>
            <t-button size="small" theme="primary" :disabled="!batch.items.every((item) => item.status === 'approved')" @click="store.closeBatch(batch.id)">结束批次</t-button>
          </header>
          <div class="batch-progress">
            <div class="batch-progress-bar"><i :style="{ width: `${batch.items.length ? (counts(batch).approved / batch.items.length) * 100 : 0}%` }" /></div>
            <span>{{ counts(batch).approved }}/{{ batch.items.length }} 通过 · {{ counts(batch).returned }} 退回 · {{ counts(batch).pending }} 待审</span>
          </div>
          <p v-if="!batch.items.every((item) => item.status === 'approved')" class="batch-hint">全部条目通过后才能结束批次；已通过的条目若被再次编辑，只会把该条标回待审。</p>
          <div class="batch-items">
            <div v-for="(item, index) in batch.items" :key="item.entryId" class="batch-item" :class="item.status">
              <div class="batch-item-main">
                <span class="item-index">{{ String(index + 1).padStart(2, '0') }}</span>
                <button class="item-head" @click="jumpTo(item.entryId)">{{ entryOf(item.entryId)?.headword || '未命名词条' }}</button>
                <span class="item-pron">[{{ entryOf(item.entryId)?.pronunciation || '音标待补' }}]</span>
                <t-tag size="small" variant="light" :theme="itemMeta[item.status].theme">{{ itemMeta[item.status].label }}</t-tag>
              </div>
              <div class="batch-item-sub">
                <span v-if="item.status === 'returned'" class="return-note">退回原因：{{ item.note || '未填写' }}</span>
                <span v-else-if="item.status === 'approved'" class="approved-hint">内容再变更会自动标回待审</span>
                <span v-else class="pending-hint">等待主审确认</span>
                <time v-if="item.decidedAt">{{ new Date(item.decidedAt).toLocaleString('zh-CN') }}</time>
                <div class="item-actions">
                  <t-button size="small" theme="success" :disabled="item.status === 'approved'" @click="store.decideBatchItem(batch.id, item.entryId, 'approved')">通过</t-button>
                  <t-button size="small" theme="danger" variant="outline" @click="openReturn(item.entryId)">退回</t-button>
                </div>
              </div>
              <div v-if="returnFor === item.entryId" class="return-box">
                <t-textarea v-model="returnNote" :autosize="{ minRows: 1, maxRows: 3 }" placeholder="退回原因（可选），会写入批次记录" />
                <t-button size="small" theme="danger" @click="submitReturn(batch.id, item.entryId)">确认退回</t-button>
                <t-button size="small" variant="text" @click="returnFor = ''">取消</t-button>
              </div>
            </div>
            <t-empty v-if="!batch.items.length" description="批次内词条已被全部移除，可直接结束批次" />
          </div>
        </article>
        <t-empty v-if="!store.openBatches.length && !creating" description="还没有进行中的审校批次，点击右上角“发起批次”开始" />
      </div>

      <section v-if="store.closedBatches.length" class="batch-closed">
        <h3>已结束批次</h3>
        <article v-for="batch in store.closedBatches" :key="batch.id" class="batch-card closed">
          <header class="batch-card-head">
            <div>
              <div class="batch-title"><strong>{{ batch.title }}</strong><t-tag size="small" theme="success" variant="light">已结束</t-tag></div>
              <span class="batch-meta">{{ batch.reviewer }} · 结束于 {{ batch.closedAt ? new Date(batch.closedAt).toLocaleString('zh-CN') : '—' }}</span>
            </div>
            <span class="closed-count">{{ batch.items.length }} 条全部通过</span>
          </header>
          <div class="closed-items">
            <span v-for="item in batch.items" :key="item.entryId" class="closed-chip">{{ entryOf(item.entryId)?.headword ?? '已删除词条' }}</span>
          </div>
        </article>
      </section>
    </div>
  </t-drawer>
</template>
