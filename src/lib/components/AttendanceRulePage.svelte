<script lang="ts">
	import { onMount } from 'svelte';
	// ─── Types (exported so other modules can import them) ────────────────────
	interface AttendanceRuleResponse {
		office_name: string;
		office_latitude: number;
		office_longitude: number;
		clock_in_start: string; // e.g. "07:15:00" or "07:15:00.000Z"
		clock_in_end: string;
		clock_out_time: string;
	}

	type AttendanceRuleUpdateRequest = AttendanceRuleResponse;

	// ─── Internal form state (time fields stored as HH:MM for <input type="time">) ─
	type FormState = {
		office_name: string;
		office_latitude: string;
		office_longitude: string;
		clock_in_start: string;
		clock_in_end: string;
		clock_out_time: string;
	};

	let form = $state<FormState>({
		office_name: '',
		office_latitude: '',
		office_longitude: '',
		clock_in_start: '',
		clock_in_end: '',
		clock_out_time: ''
	});

	let loading = $state(true);
	let saving = $state(false);
	let fetchError = $state<string | null>(null);
	let saveError = $state<string | null>(null);
	let saveSuccess = $state(false);
	let validationErrors = $state<string[]>([]);

	// ─── Time helpers ─────────────────────────────────────────────────────────

	/**
	 * Parse API time string (e.g. "07:15:34.298Z" or "07:15:00") → "HH:MM"
	 * for <input type="time"> (zona WIB — dianggap backend sudah kirim waktu lokal).
	 */
	function parseTimeToInput(timeStr: string): string {
		if (!timeStr) return '';
		// Hapus suffix 'Z' atau '.000Z' lalu ambil dua segmen pertama
		const cleaned = timeStr.replace(/\..*Z?$/, '').replace('Z', '');
		const parts = cleaned.split(':');
		if (parts.length >= 2) {
			return `${parts[0].padStart(2, '0')}:${parts[1].padStart(2, '0')}`;
		}
		return '';
	}

	/**
	 * Konversi nilai "HH:MM" dari <input type="time"> → "HH:MM:SS" untuk dikirim ke API.
	 */
	function inputToApiTime(timeStr: string): string {
		if (!timeStr) return '00:00:00';
		return `${timeStr}:00`;
	}

	// ─── Data fetching ────────────────────────────────────────────────────────

	/** Ambil konfigurasi aturan absensi dari GET /api/v1/attendance-admin/rule */
	async function fetchRule() {
		loading = true;
		fetchError = null;

		try {
			const res = await fetch(
				'https://presence-api-production-988a.up.railway.app/api/v1/attendance-admin/rule'
			);
			if (!res.ok) throw new Error(`Server error: ${res.status} ${res.statusText}`);

			const data: AttendanceRuleResponse = await res.json();

			// Isi form dengan data dari API; konversi tipe numerik & waktu
			form.office_name = data.office_name ?? '';
			form.office_latitude = String(data.office_latitude ?? '');
			form.office_longitude = String(data.office_longitude ?? '');
			form.clock_in_start = parseTimeToInput(data.clock_in_start);
			form.clock_in_end = parseTimeToInput(data.clock_in_end);
			form.clock_out_time = parseTimeToInput(data.clock_out_time);
		} catch (err) {
			fetchError = err instanceof Error ? err.message : 'Gagal mengambil data aturan absensi.';
		} finally {
			loading = false;
		}
	}

	// ─── Form submission ──────────────────────────────────────────────────────

	/** Validasi, lalu kirim PUT /api/v1/attendance-admin/rule */
	async function handleSubmit(e: Event) {
		e.preventDefault();
		validationErrors = [];
		saveError = null;
		saveSuccess = false;

		// Validasi client-side: semua field wajib diisi
		// Cast ke string dulu supaya aman jika ada nilai number/null dari sumber lain (SSR/hydration)
		if (!String(form.office_name ?? '').trim())
			validationErrors.push('Nama Kantor wajib diisi.');
		if (!String(form.office_latitude ?? '').trim())
			validationErrors.push('Latitude wajib diisi.');
		if (!String(form.office_longitude ?? '').trim())
			validationErrors.push('Longitude wajib diisi.');
		if (!form.clock_in_start) validationErrors.push('Clock In Start wajib diisi.');
		if (!form.clock_in_end) validationErrors.push('Clock In End wajib diisi.');
		if (!form.clock_out_time) validationErrors.push('Clock Out wajib diisi.');

		// Validasi: clock_in_start harus lebih awal dari clock_in_end
		if (form.clock_in_start && form.clock_in_end && form.clock_in_start >= form.clock_in_end) {
			validationErrors.push('Clock In Start harus lebih awal dari Clock In End.');
		}

		if (validationErrors.length > 0) return;

		saving = true;
		try {
			// Susun body request; konversi HH:MM → HH:MM:SS & string → number
			const body: AttendanceRuleUpdateRequest = {
				office_name: form.office_name.trim(),
				office_latitude: parseFloat(form.office_latitude),
				office_longitude: parseFloat(form.office_longitude),
				clock_in_start: inputToApiTime(form.clock_in_start),
				clock_in_end: inputToApiTime(form.clock_in_end),
				clock_out_time: inputToApiTime(form.clock_out_time)
			};

			const res = await fetch(
				'https://presence-api-production-988a.up.railway.app/api/v1/attendance-admin/rule',
				{
					method: 'PUT',
					headers: { 'Content-Type': 'application/json' },
					body: JSON.stringify(body)
				}
			);

			if (!res.ok) {
				const errData = await res.json().catch(() => ({}));
				throw new Error(
					(errData as { message?: string })?.message ?? `Server error: ${res.status}`
				);
			}

			saveSuccess = true;
			// Sembunyikan banner sukses otomatis setelah 4 detik
			setTimeout(() => {
				saveSuccess = false;
			}, 4000);
		} catch (err) {
			saveError = err instanceof Error ? err.message : 'Gagal menyimpan perubahan.';
		} finally {
			saving = false;
		}
	}

	// Panggil GET saat komponen pertama kali dimount
	onMount(() => {
		fetchRule();
	});
</script>

<div class="min-h-screen bg-gray-50 px-4 py-10">
	<div class="mx-auto max-w-xl">
		<!-- Page Header -->
		<div class="mb-8">
			<h1 class="text-2xl font-bold text-gray-900">Aturan Absensi</h1>
			<p class="mt-1 text-sm text-gray-500">
				Kelola konfigurasi lokasi kantor dan jam absensi karyawan.
			</p>
		</div>

		<!-- Loading state -->
		{#if loading}
			<div class="flex items-center justify-center rounded-xl bg-white p-12 shadow">
				<div class="flex flex-col items-center gap-3">
					<svg
						class="h-8 w-8 animate-spin text-blue-500"
						xmlns="http://www.w3.org/2000/svg"
						fill="none"
						viewBox="0 0 24 24"
					>
						<circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"
						></circle>
						<path
							class="opacity-75"
							fill="currentColor"
							d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"
						></path>
					</svg>
					<span class="text-sm text-gray-500">Memuat data aturan absensi…</span>
				</div>
			</div>

			<!-- Fetch error state -->
		{:else if fetchError}
			<div class="rounded-xl border border-red-200 bg-red-50 p-6 shadow">
				<div class="flex items-start gap-3">
					<svg
						class="mt-0.5 h-5 w-5 shrink-0 text-red-500"
						xmlns="http://www.w3.org/2000/svg"
						viewBox="0 0 20 20"
						fill="currentColor"
					>
						<path
							fill-rule="evenodd"
							d="M10 18a8 8 0 100-16 8 8 0 000 16zm-.75-4.75a.75.75 0 001.5 0v-4.5a.75.75 0 00-1.5 0v4.5zm.75-7.5a1 1 0 110 2 1 1 0 010-2z"
							clip-rule="evenodd"
						/>
					</svg>
					<div>
						<p class="font-semibold text-red-700">Gagal memuat data</p>
						<p class="mt-1 text-sm text-red-600">{fetchError}</p>
					</div>
				</div>
				<button
					onclick={fetchRule}
					class="mt-4 rounded-lg bg-red-600 px-4 py-2 text-sm font-medium text-white transition hover:bg-red-700 focus:ring-2 focus:ring-red-400 focus:outline-none"
				>
					Coba Lagi
				</button>
			</div>

			<!-- Form -->
		{:else}
			<!-- Success banner -->
			{#if saveSuccess}
				<div
					class="mb-4 flex items-center gap-3 rounded-lg border border-green-200 bg-green-50 px-4 py-3 text-green-800"
				>
					<svg
						class="h-5 w-5 shrink-0 text-green-500"
						xmlns="http://www.w3.org/2000/svg"
						viewBox="0 0 20 20"
						fill="currentColor"
					>
						<path
							fill-rule="evenodd"
							d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.857-9.809a.75.75 0 00-1.214-.882l-3.483 4.79-1.88-1.88a.75.75 0 10-1.06 1.061l2.5 2.5a.75.75 0 001.137-.089l4-5.5z"
							clip-rule="evenodd"
						/>
					</svg>
					<span class="text-sm font-medium">Aturan absensi berhasil disimpan.</span>
				</div>
			{/if}

			<form onsubmit={handleSubmit} class="rounded-xl bg-white p-6 shadow" novalidate>
				<!-- Validation errors -->
				{#if validationErrors.length > 0}
					<div class="mb-6 rounded-lg border border-yellow-200 bg-yellow-50 px-4 py-3">
						<p class="mb-1 text-sm font-semibold text-yellow-800">
							Mohon perbaiki kesalahan berikut:
						</p>
						<ul class="list-inside list-disc space-y-0.5">
							{#each validationErrors as err}
								<li class="text-sm text-yellow-700">{err}</li>
							{/each}
						</ul>
					</div>
				{/if}

				<!-- ── Informasi Kantor ───────────────────────────────────────── -->
				<fieldset class="mb-6">
					<legend class="mb-3 text-xs font-semibold tracking-widest text-gray-400 uppercase">
						Informasi Kantor
					</legend>
					<div class="space-y-4">
						<!-- Nama Kantor -->
						<div>
							<label for="office_name" class="mb-1 block text-sm font-medium text-gray-700">
								Nama Kantor <span class="text-red-500">*</span>
							</label>
							<input
								id="office_name"
								type="text"
								bind:value={form.office_name}
								placeholder="Contoh: Kantor Pusat Jakarta"
								class="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm text-gray-900 placeholder-gray-400 transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none"
							/>
						</div>

						<!-- Koordinat (grid 2 kolom) -->
						<div class="grid grid-cols-2 gap-4">
							<!-- Latitude -->
							<div>
								<label for="office_latitude" class="mb-1 block text-sm font-medium text-gray-700">
									Latitude <span class="text-red-500">*</span>
								</label>
								<input
									id="office_latitude"
									type="number"
									step="any"
									bind:value={form.office_latitude}
									placeholder="-6.2088"
									class="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm text-gray-900 placeholder-gray-400 transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none"
								/>
							</div>
							<!-- Longitude -->
							<div>
								<label for="office_longitude" class="mb-1 block text-sm font-medium text-gray-700">
									Longitude <span class="text-red-500">*</span>
								</label>
								<input
									id="office_longitude"
									type="number"
									step="any"
									bind:value={form.office_longitude}
									placeholder="106.8456"
									class="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm text-gray-900 placeholder-gray-400 transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none"
								/>
							</div>
						</div>
					</div>
				</fieldset>

				<hr class="mb-6 border-gray-100" />

				<!-- ── Jam Absensi ────────────────────────────────────────────── -->
				<fieldset class="mb-6">
					<legend class="mb-3 text-xs font-semibold tracking-widest text-gray-400 uppercase">
						Jam Absensi (WIB)
					</legend>
					<div class="grid grid-cols-1 gap-4 sm:grid-cols-3">
						<!-- Clock In Start -->
						<div>
							<label for="clock_in_start" class="mb-1 block text-sm font-medium text-gray-700">
								Clock In Start <span class="text-red-500">*</span>
							</label>
							<input
								id="clock_in_start"
								type="time"
								bind:value={form.clock_in_start}
								class="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm text-gray-900 transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none"
							/>
							<p class="mt-1 text-xs text-gray-400">Awal batas masuk</p>
						</div>

						<!-- Clock In End -->
						<div>
							<label for="clock_in_end" class="mb-1 block text-sm font-medium text-gray-700">
								Clock In End <span class="text-red-500">*</span>
							</label>
							<input
								id="clock_in_end"
								type="time"
								bind:value={form.clock_in_end}
								class="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm text-gray-900 transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none"
							/>
							<p class="mt-1 text-xs text-gray-400">Batas akhir masuk</p>
						</div>

						<!-- Clock Out -->
						<div>
							<label for="clock_out_time" class="mb-1 block text-sm font-medium text-gray-700">
								Clock Out <span class="text-red-500">*</span>
							</label>
							<input
								id="clock_out_time"
								type="time"
								bind:value={form.clock_out_time}
								class="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm text-gray-900 transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200 focus:outline-none"
							/>
							<p class="mt-1 text-xs text-gray-400">Waktu pulang</p>
						</div>
					</div>
				</fieldset>

				<!-- Save error -->
				{#if saveError}
					<div
						class="mb-4 flex items-start gap-2 rounded-lg border border-red-200 bg-red-50 px-4 py-3 text-sm text-red-700"
					>
						<svg
							class="mt-0.5 h-4 w-4 shrink-0"
							xmlns="http://www.w3.org/2000/svg"
							viewBox="0 0 20 20"
							fill="currentColor"
						>
							<path
								fill-rule="evenodd"
								d="M10 18a8 8 0 100-16 8 8 0 000 16zm-.75-4.75a.75.75 0 001.5 0v-4.5a.75.75 0 00-1.5 0v4.5zm.75-7.5a1 1 0 110 2 1 1 0 010-2z"
								clip-rule="evenodd"
							/>
						</svg>
						<span>{saveError}</span>
					</div>
				{/if}

				<!-- Submit button -->
				<div class="flex justify-end">
					<button
						type="submit"
						disabled={saving}
						class="inline-flex items-center gap-2 rounded-lg bg-blue-600 px-5 py-2.5 text-sm font-semibold text-white transition hover:bg-blue-700 focus:ring-2 focus:ring-blue-400 focus:outline-none disabled:cursor-not-allowed disabled:opacity-60"
					>
						{#if saving}
							<svg
								class="h-4 w-4 animate-spin"
								xmlns="http://www.w3.org/2000/svg"
								fill="none"
								viewBox="0 0 24 24"
							>
								<circle
									class="opacity-25"
									cx="12"
									cy="12"
									r="10"
									stroke="currentColor"
									stroke-width="4"
								></circle>
								<path
									class="opacity-75"
									fill="currentColor"
									d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"
								></path>
							</svg>
							Menyimpan…
						{:else}
							Simpan
						{/if}
					</button>
				</div>
			</form>
		{/if}
	</div>
</div>
