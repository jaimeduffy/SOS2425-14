<script>
	// @ts-nocheck
	import { onMount } from 'svelte';
	import { dev } from '$app/environment';
	import { fade } from 'svelte/transition';
	//import Highcharts from 'highcharts';

	let DEVEL_HOST = 'http://localhost:16078';
	let BASE_API = '/api/v1/cybercrime-data';

	if (dev) {
		BASE_API = DEVEL_HOST + BASE_API;
	}

	let myData = [];
	let years = [];
	let loadingData = true;
	let errorMessage = '';

	let showChart = false;

	// Lista de comunidades autónomas españolas
	const communities = [
		'Andalucia',
		'Aragon',
		'Asturias',
		'Baleares',
		'Canarias',
		'Cantabria',
		'Castilla-La Mancha',
		'Castilla y Leon',
		'Cataluña',
		'Comunitat Valenciana',
		'Extremadura',
		'Galicia',
		'Madrid',
		'Murcia',
		'Navarra',
		'Pais Vasco',
		'La Rioja',
		'Ceuta y Melilla',
		'TOTAL'
	];

	let selectedBarYear = '';
	let selectedCommunity = '';

	// Función que se ejecuta cuando se cambia la comunidad seleccionada
	function handleCommunityChange() {
		errorMessage = '';
		showChart = true;
		getData();
	}

	async function getData() {
		loadingData = true;
		errorMessage = '';

		// Construir la URL con el parámetro de la comunidad seleccionada
		const API = `${BASE_API}?autonomous_community=${encodeURIComponent(selectedCommunity)}`;

		try {
			console.log(`Solicitando datos a: ${API}`);
			const res = await fetch(API, { method: 'GET' });
			if (!res.ok) {
				throw new Error(`Error en la respuesta: ${res.status}`);
			}

			const data = await res.json();
			myData = data;
			console.log(`Datos recibidos: ${myData.length} registros para ${selectedCommunity}`);

			// Extraer años y tipos de delitos únicos
			years = [...new Set(myData.map((item) => item.year))].sort();

			renderChart();
			renderBarChart(); // Añadido para renderizar el gráfico de barras
		} catch (error) {
			console.error(`ERROR al obtener datos de ${API}: ${error}`);
			// @ts-ignore
			errorMessage = `Error al cargar datos: ${error.message}`;
		} finally {
			loadingData = false;
		}
	}

	function renderChart() {
		if (myData.length === 0) {
			errorMessage = 'No hay datos disponibles para la comunidad seleccionada';
			return;
		}

		const criminalOfense = [];
		const cybersecurity = [];
		const arrestedInvestigated = [];

		years.forEach((year) => {
			const record = myData.find((r) => String(r.year) === String(year));
			criminalOfense.push(record?.criminal_ofense ?? 0);
			cybersecurity.push(record?.cybersecurity ?? 0);
			arrestedInvestigated.push(record?.arrested_investigated ?? 0);
		});

		Highcharts.chart('container', {
			chart: {
				type: 'area', // Cambiar el tipo de gráfico a 'area'
				backgroundColor: '#052a42',
				spacing: [30, 25, 30, 25]
			},
			title: {
				text: 'Evolución de las Tasas de Delitos',
				align: 'left',
				style: {
					color: '#fff',
					fontWeight: '800'
				}
			},
			subtitle: {
				text: `Delitos en ${selectedCommunity}`,
				align: 'left',
				style: {
					color: '#fff',
					fontWeight: 'bold'
				}
			},
			yAxis: {
				title: {
					text: 'Número de Delitos',
					style: { color: '#fff', fontWeight: '800' }
				},
				labels: { style: { color: '#fff' } },
				gridLineColor: '#ffffff33'
			},
			xAxis: {
				categories: years,
				title: {
					text: 'AÑO',
					style: { color: '#fff', fontWeight: '800' }
				},
				labels: { style: { color: '#fff' } }
			},
			legend: {
				layout: 'vertical',
				align: 'right',
				verticalAlign: 'middle',
				itemStyle: { color: '#fff' }
			},
			plotOptions: {
				area: {
					fillOpacity: 0.5,
					marker: { enabled: false }
				},
				series: {
					label: { connectorAllowed: false }
				}
			},
			series: [
				{ name: 'Ofensa criminal', data: criminalOfense, color: '#36A2EB' },
				{ name: 'Ciberseguridad', data: cybersecurity, color: '#FFCE56' },
				{ name: 'Investigados Arrestados', data: arrestedInvestigated, color: '#FF6384' }
			],
			responsive: {
				rules: [{
					condition: { maxWidth: 500 },
					chartOptions: {
						legend: {
							layout: 'horizontal',
							align: 'center',
							verticalAlign: 'bottom'
						}
					}
				}]
			}
		});
	}

	let barChartInstance;

	function renderBarChart() {
		const yearToShow = selectedBarYear || Math.max(...years);
		const record = myData.find((r) => Number(r.year) === Number(yearToShow));

		if (!record) return;

		const ctx = document.getElementById('barChart')?.getContext('2d');
		if (!ctx) return;

		// Destruir el gráfico anterior si existe
		if (barChartInstance) {
			barChartInstance.destroy();
			barChartInstance = null;
		}

		barChartInstance = new Chart(ctx, {
			type: 'pie', // Cambiar a tipo 'pie' (gráfico circular)
			data: {
				labels: ['Ofensa Criminal', 'Ciberseguridad', 'Investigados arrestados'],
				datasets: [{
					label: `${selectedCommunity} - ${yearToShow}`,
					data: [record.criminal_ofense ?? 0, record.cybersecurity ?? 0, record.arrested_investigated ?? 0],
					backgroundColor: ['#36A2EB', '#FFCE56', '#FF6384']
				}]
			},
			options: {
				responsive: true,
				plugins: {
					legend: {
						labels: { color: '#fff' }
					},
					title: {
						display: true,
						text: 'Distribución de delitos por nivel de cibercriminalidad',
						color: '#fff',
						font: { size: 20, weight: 'bold', family: 'Arial' }
					}
				},
				scales: {
					x: { ticks: { color: '#fff' }, title: { display: true, text: 'Nivel de cibercriminalidad', color: '#fff' } },
					y: { ticks: { color: '#fff' }, title: { display: true, text: 'Número de Delitos', color: '#fff' } }
				}
			}
		});
	}

	// Esta función carga los scripts en vez de svelte:head
	// @ts-ignore
	function loadScript(src) {
		return new Promise((resolve, reject) => {
			const existing = document.querySelector(`script[src="${src}"]`);
			// @ts-ignore
			if (existing) return resolve();

			const script = document.createElement('script');
			script.src = src;
			script.onload = resolve;
			script.onerror = reject;
			document.head.appendChild(script);
		});
	}

	onMount(async () => {
		try {
			// HighCharts
			await loadScript('https://code.highcharts.com/highcharts.js');
			await loadScript('https://code.highcharts.com/modules/series-label.js');
			await loadScript('https://code.highcharts.com/modules/exporting.js');
			await loadScript('https://code.highcharts.com/modules/export-data.js');
			await loadScript('https://code.highcharts.com/modules/accessibility.js');

			// Charts.js
			await loadScript('https://cdn.jsdelivr.net/npm/chart.js');

			await getData();
		} catch (error) {
			errorMessage = `Error cargando scripts de HighCharts: ${error}`;
			console.error(error);
		}
	});
</script>

<svelte:head>
	<title>Graphs - Cybercrime Data</title>
	<meta charset="UTF-8" />
	<meta name="viewport" content="width=device-width, initial-scale=1.0" />
</svelte:head>

<div class="wrapper">
	<div class="container">
		<h2>Gráficas de cybercrime-data</h2>
		<hr style="width: 55em; animation: loadHrGraph 1s; transition: all 0.3s ease;" />

		<!-- Selector de comunidad -->
		<div class="graph-container" style="background: none; box-shadow: none; padding: 0.5em 0;">
			<h3>Selecciona la comunidad autónoma:</h3>
			<select
				class="custom-select"
				bind:value={selectedCommunity}
				on:change={handleCommunityChange}
				style="width: 150%;"
			>
				<option disabled value="">Selecciona una comunidad</option>
				{#each communities as community}
					<option value={community}>{community}</option>
				{/each}
			</select>
		</div>

		{#if showChart}
			<div transition:fade={{ duration: 400 }}>
				<div class="article">
					<h3 style="font-size: 1.5em; text-transform: none;">
						Evolución temporal de las tasas de delitos por nivel de cibercriminalidad
					</h3>
					<p>
						Visualiza cómo han cambiado las tasas de delitos a lo largo del tiempo y entre distintas comunidades
						autónomas.
					</p>
					<br />
					<figure class="highcharts-figure">
						<div id="container">
							{#if loadingData}
								<p>Cargando Datos...</p>
							{/if}
							{#if errorMessage}
								<p style="background-color: rgb(247, 111, 111); padding: 10px 20px;">
									{errorMessage}
								</p>
							{/if}
						</div>
					</figure>
				</div>
				<div class="article" style="margin-top: 0;">
					<h3 style="font-size: 1.5em; text-transform: none;">
						Distribución de delitos por nivel de cibercriminalidad en el año seleccionado
					</h3>
					<p>
						Compara el número de delitos en los distintos tipos de cibercriminalidad
						durante un año específico en la comunidad autónoma seleccionada, permitiendo analizar la
						distribución entre Ofensas criminales, ciberseguridad y Investigados arrestado.
					</p>
					<div
						class="graph-container"
						style="background: none; box-shadow: none; padding: 0.5em 0; margin-top: -1em; font-size: 90%;"
					>
						<h3 style="font-size: 1.2em; text-transform: none; font-weight: 500;">
							Selecciona el Año:
						</h3>
						<select bind:value={selectedBarYear} on:change={renderBarChart} class="custom-select">
							<option disabled value="">Selecciona un año</option>
							{#each years as year}
								<option value={year}>{year}</option>
							{/each}
						</select>
					</div>
					{#if selectedBarYear}
						<figure class="chartjs-figure" transition:fade>
							<canvas id="barChart"></canvas>
						</figure>
					{/if}
				</div>
			</div>
		{/if}
	</div>
</div>

<style>
	.highcharts-figure,
	.highcharts-data-table table {
		min-width: 45em;
		max-width: 45em;
		margin: 1em auto;
		box-shadow: 0 0 20px rgba(0, 0, 0, 0.459);
	}
	.chartjs-figure {
		min-width: 45em;
		max-width: 45em;
		margin: 2em auto;
		box-shadow: 0 0 20px rgba(0, 0, 0, 0.473);
		padding: 1em;
		background: #052a42;
		border-radius: 10px;
	}
</style>
