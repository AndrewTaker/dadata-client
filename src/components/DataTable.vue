<script setup lang="ts">
import { computed, ref } from 'vue';
import type { Suggestion } from '../composables/useDadata';
const props = defineProps({
	data: {
		type: Array as () => Suggestion[],
		required: true,
	},
});

const availableTags = [
	{ tag: 'полное наименование', label: 'data.name.full_with_opf' },
	{ tag: 'краткое наименование', label: 'data.name.short_with_opf' },
	{ tag: 'окато', label: 'data.okato' },
	{ tag: 'октмо', label: 'data.oktmo' },
	{ tag: 'окпо', label: 'data.okpo' },
	{ tag: 'окогу', label: 'data.okogu' },
	{ tag: 'окфс', label: 'data.okfs' },
	{ tag: 'оквэд', label: 'data.okved' },
	{ tag: 'количество филиалов', label: 'data.branch_count' },
	{ tag: 'тип подразделения', label: 'data.branch_type' },
	{ tag: 'инн', label: 'data.inn' },
	{ tag: 'кпп', label: 'data.kpp' },
	{ tag: 'адрес', label: 'data.address.unrestricted_value' },
    { tag: 'руководитель', label: 'data.management.name' },
    { tag: 'статус', label: 'data.state.status' },
];

const selectedTags = ref<string[]>(['полное наименование', 'инн']);
const sortKey = ref<string>('');
const sortOrder = ref<'asc' | 'desc' | ''>('');
const selectedColumns = computed(() =>
	availableTags
		.filter(tag => selectedTags.value.includes(tag.tag))
		.map(tag => tag.label)
);

const toggleTag = (tag: string) => {
	selectedTags.value.includes(tag)
		? selectedTags.value.splice(selectedTags.value.indexOf(tag), 1)
		: selectedTags.value.push(tag);
};


const getColumnValue = (row: any, column: string) => {
	return column.split('.').reduce((acc, key) => acc?.[key] ?? '', row);
};

const getTagLabel = (columnPath: string) => {
	const tag = availableTags.find(tag => tag.label === columnPath);
	return tag ? tag.tag : columnPath;
};
const escapeCsvValue = (value: string) => {
	if (typeof value !== 'string') value = String(value);
	return `"${value.replace(/"/g, '""')}"`;
};

const csvContent = computed(() => {
	const header = selectedColumns.value.map(escapeCsvValue).join(',');
	const rows = props.data.map(row =>
		selectedColumns.value
			.map(column => escapeCsvValue(getColumnValue(row, column)))
			.join(',')
	);
	return `data:text/csv;charset=utf-8,${header}\n${rows.join('\n')}`;
});

const downloadCSV = () => {
	const encodedUri = encodeURI(csvContent.value);
	const link = document.createElement('a');
	link.setAttribute('href', encodedUri);
	link.setAttribute('download', 'data.csv');
	link.click();
};

const sortTable = (key: string) => {
	if (sortKey.value === key) {
		sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc';
	} else {
		sortKey.value = key;
		sortOrder.value = 'asc';
	}
};

const searchQuery = ref<string>('');
const sortedData = computed(() => {
	let filteredData = props.data;

	if (searchQuery.value) {
		filteredData = filteredData.filter(row =>
			selectedColumns.value.some(column =>
				String(getColumnValue(row, column)).toLowerCase().includes(searchQuery.value.toLowerCase())
			)
		);
	}

	if (sortKey.value && sortOrder.value) {
		filteredData = [...filteredData].sort((a, b) => {
			const valueA = getColumnValue(a, sortKey.value);
			const valueB = getColumnValue(b, sortKey.value);

			if (valueA < valueB) return sortOrder.value === 'asc' ? -1 : 1;
			if (valueA > valueB) return sortOrder.value === 'asc' ? 1 : -1;
			return 0;
		});
	}

	return filteredData;
});
</script>

<template>
	<div class="w-full p-4">
		<div class="flex flex-col sm:flex-row justify-between items-start sm:items-center mb-6 gap-4">
			<div class="flex flex-wrap gap-2">
				<span v-for="tag in availableTags" :key="tag.tag" @click="toggleTag(tag.tag)"
					class="px-3 py-1 text-sm font-medium rounded-full cursor-pointer transition" :class="{
						'bg-blue-100 text-blue-800 hover:bg-blue-200': !selectedTags.includes(tag.tag),
						'bg-blue-600 text-white hover:bg-blue-700': selectedTags.includes(tag.tag),
					}">
					{{ tag.tag }}
				</span>
			</div>

			<input v-model="searchQuery" type="text" placeholder="Поиск"
				class="px-4 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500" />

			<button @click="downloadCSV"
				class="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 transition">
				Скачать CSV
			</button>
		</div>

		<div class="w-full overflow-auto rounded-lg shadow-sm border border-gray-200 max-h-[80vh]">
			<table class="w-full">
				<thead class="bg-gray-200 sticky top-0 z-10">
					<tr>
						<th v-for="(column, index) in selectedColumns" :key="index" @click="sortTable(column)"
							class="px-6 py-3 text-left text-sm font-medium text-gray-600 uppercase tracking-wider cursor-pointer hover:bg-gray-300 transition">
							{{ getTagLabel(column) }}
							<span v-if="sortKey === column">
								{{ sortOrder === 'asc' ? '▲' : '▼' }}
							</span>
						</th>
					</tr>
				</thead>

				<tbody class="divide-y divide-gray-200">
					<tr v-for="(row, rowIndex) in sortedData" :key="rowIndex" class="hover:bg-gray-50 transition">
						<td v-for="column in selectedColumns" :key="column" class="px-6 py-4 text-sm text-gray-700">
							{{ getColumnValue(row, column) }}
						</td>
					</tr>
				</tbody>
			</table>
		</div>
	</div>
</template>
