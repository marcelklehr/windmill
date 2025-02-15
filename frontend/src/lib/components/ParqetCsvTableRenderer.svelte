<script lang="ts">
	import { GridApi, createGrid, type IDatasource } from 'ag-grid-community'

	import 'ag-grid-community/styles/ag-grid.css'
	import 'ag-grid-community/styles/ag-theme-alpine.css'
	import { twMerge } from 'tailwind-merge'
	import DarkModeObserver from './DarkModeObserver.svelte'
	import { HelpersService } from '$lib/gen'
	import { enterpriseLicense, workspaceStore } from '$lib/stores'
	import { Download } from 'lucide-svelte'

	// import 'ag-grid-community/dist/styles/ag-theme-alpine-dark.css'

	let selectedRowIndex = -1
	export let s3resource: string
	export let storage: string | undefined
	export let workspaceId: string | undefined
	export let disable_download: boolean = false

	let datasource: IDatasource = {
		rowCount: 0,
		getRows: async function (params) {
			try {
				const searchCol = params.filterModel ? Object.keys(params.filterModel)?.[0] : undefined
				const requestBody = {
					workspace: workspaceId ?? $workspaceStore!,
					path: s3resource,
					offset: params.startRow,
					limit: params.endRow - params.startRow,
					sortCol: params.sortModel?.[0]?.colId,
					sortDesc: params.sortModel?.[0]?.sort == 'desc',
					searchCol: searchCol,
					searchTerm: searchCol ? params.filterModel?.[searchCol]?.filter : undefined,
					storage: storage
				}
				const csv = s3resource.endsWith('.csv')
				const res = (
					csv
						? await HelpersService.loadCsvPreview(requestBody)
						: await HelpersService.loadParquetPreview(requestBody)
				) as any
				for (let i = 0; i < res.rows.length; i++) {
					res.rows[i]['__index'] = i + params.startRow
					if (!$enterpriseLicense) {
						Object.keys(res.rows[i]).forEach((key) => {
							if (key != '__index') {
								res.rows[i][key] = 'Require EE'
							}
						})
					}
				}

				params.successCallback(res.rows)
			} catch (e) {
				console.error(e)
				params.failCallback()
			}
		}
	}

	function toggleRow(row: any) {
		if (row) {
			let rowIndex = row.rowIndex
			let data = { ...row.data }
			delete data['__index']
			if (selectedRowIndex !== rowIndex) {
				selectedRowIndex = rowIndex
			}
		}
	}

	function toggleRows(rows: any[]) {
		toggleRow(rows[0])
	}

	let eGui: HTMLDivElement

	$: eGui && mountGrid()

	async function mountGrid() {
		if (eGui) {
			const csv = s3resource.endsWith('.csv')

			const res = csv
				? await HelpersService.loadCsvPreview({
						workspace: $workspaceStore!,
						path: s3resource,
						limit: 0,
						storage: storage
				  })
				: await HelpersService.loadParquetPreview({
						workspace: $workspaceStore!,
						path: s3resource,
						limit: 0,
						storage: storage
				  })

			createGrid(
				eGui,
				{
					rowModelType: 'infinite',
					datasource,
					// @ts-ignore
					columnDefs: res.columns.map((c) => {
						return {
							field: c,
							sortable: true,
							filter: true,
							filterParams: {
								filterOptions: ['contains'],
								maxNumConditions: 1
							}
						}
					}),
					pagination: false,
					// defaultColDef: {
					// 	flex: 1
					// },
					suppressColumnMoveAnimation: true,
					rowSelection: 'multiple',
					rowMultiSelectWithClick: true,
					suppressRowDeselection: true,

					onSelectionChanged: (e) => {
						onSelectionChanged(e.api)
					},
					getRowId: (data) => {
						return (data as any).data['__index']
					}
				},
				{}
			)
		}
	}

	function onSelectionChanged(api: GridApi<any>) {
		const rows = api.getSelectedNodes()
		if (rows != undefined) {
			toggleRows(rows)
		}
	}

	let darkMode: boolean = false
</script>

<DarkModeObserver bind:darkMode />

<div class={twMerge('mt-2  flex flex-col h-full min-h-[600px]')}>
	{#if !disable_download && !s3resource.endsWith('.csv')}
		<a
			target="_blank"
			href="/api/w/{workspaceId}/job_helpers/download_s3_parquet_file_as_csv?file_key={s3resource}{storage
				? `&storage=${storage}`
				: ''}"
			class="text-secondary w-full text-right underline text-2xs whitespace-nowrap"
			><div class="flex flex-row-reverse gap-2 items-center"><Download size={12} /> CSV</div></a
		>
	{/if}

	<div
		class="ag-theme-alpine shadow-sm h-full"
		class:ag-theme-alpine-dark={darkMode}
		style="height: 600px;"
	>
		<div bind:this={eGui} style="height:100%; " />
	</div>
</div>

<!-- <div class="flex gap-1 absolute bottom-1 right-2 text-sm text-secondary"
	>{firstRow}{'->'}{lastRow + 1} of {datasource?.rowCount} rows</div
> -->
