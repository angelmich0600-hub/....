<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Reporte Stock Corregido</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.3.2/papaparse.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;800&display=swap');
        :root { --accent: #2563eb; --danger: #ef4444; }
        body { font-family: 'Inter', sans-serif; margin: 0; padding: 10px; background: #fff; color: #000; }
        
        .sync-panel {
            position: fixed; top: 10px; right: 10px; background: white; 
            padding: 8px; border-radius: 8px; border: 1px solid #ddd;
            display: flex; gap: 8px; align-items: center; box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            z-index: 1000;
        }
        .status-dot { width: 8px; height: 8px; border-radius: 50%; background: #ccc; }
        .online { background: #10b981; }

        .report-header { 
            border-bottom: 2px solid #000; 
            margin-bottom: 15px; 
            display: flex; 
            justify-content: space-between; 
            align-items: flex-end;
            padding-bottom: 5px;
        }
        
        #inventoryOutput { 
            display: grid; 
            grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); 
            gap: 10px; 
        }
        
        .brand-block { border: 1px solid #eee; break-inside: avoid; margin-bottom: 5px; }
        .brand-header { 
            background: #f3f4f6; padding: 3px 8px; font-size: 10px; font-weight: 800; 
            display: flex; justify-content: space-between; border-bottom: 1px solid #ddd;
            text-transform: uppercase;
        }
        
        table { width: 100%; border-collapse: collapse; }
        td { padding: 3px 8px; font-size: 10px; border-bottom: 1px solid #f9f9f9; line-height: 1.1; }
        .qty { text-align: right; font-weight: 700; color: var(--accent); width: 30px; border-left: 1px solid #f0f0f0; }

        .btn { border: none; padding: 5px 10px; border-radius: 4px; cursor: pointer; font-weight: 700; font-size: 10px; color: white; }
        .btn-blue { background: #4f46e5; }
        .btn-green { background: #10b981; }

        /* Alerta si faltan datos */
        #debugInfo { background: #fef2f2; border: 1px solid #fee2e2; color: #991b1b; padding: 8px; font-size: 10px; margin-bottom: 10px; display: none; }

        @media print {
            .sync-panel, #debugInfo { display: none !important; }
            #inventoryOutput { display: block; column-count: 3; column-gap: 10px; }
            .brand-block { display: inline-block; width: 100%; margin-bottom: 8px; }
        }
    </style>
</head>
<body>

<div class="sync-panel">
    <span id="statusIcon" class="status-dot"></span>
    <span id="statusText" style="font-size: 10px; font-weight: 600;">Sincronizando...</span>
    <button class="btn btn-blue" onclick="fetchData()">🔄</button>
    <button class="btn btn-green" onclick="window.print()">🖨️ IMPRIMIR</button>
</div>

<div class="report-container">
    <div id="debugInfo"></div>
    <div class="report-header">
        <div>
            <h1 style="margin:0; font-size: 20px; font-weight: 900;">STOCK DISPONIBLE</h1>
            <div id="totalGlobal" style="color: var(--accent); font-weight: 700; font-size: 12px;">Cargando...</div>
        </div>
        <div style="text-align: right; font-size: 9px; color: #666;">
            <div id="currentDate"></div>
            <div>Actualización en tiempo real</div>
        </div>
    </div>

    <div id="inventoryOutput"></div>
</div>

<script>
    const SHEET_URL = "https://docs.google.com/spreadsheets/d/10s3oYdLwVT_TLuNgRP14l6Rk_G0sHdQOWi_PEFo3grc/gviz/tq?tqx=out:csv";

    async function fetchData() {
        const statusIcon = document.getElementById('statusIcon');
        const statusText = document.getElementById('statusText');
        statusIcon.classList.remove('online');

        Papa.parse(SHEET_URL, {
            download: true,
            header: false,
            skipEmptyLines: true,
            complete: function(results) {
                processData(results.data);
                statusIcon.classList.add('online');
                statusText.innerText = "Sincronizado";
                document.getElementById('currentDate').innerText = new Date().toLocaleString('es-ES');
            }
        });
    }

    function processData(data) {
        const output = document.getElementById('inventoryOutput');
        const debug = document.getElementById('debugInfo');
        output.innerHTML = '';
        debug.innerHTML = '';
        debug.style.display = 'none';

        const brands = {};
        let grandTotal = 0;
        let rowsSkipped = 0;

        data.forEach((row, index) => {
            // CORRECCIÓN 1: No saltar la fila 0 automáticamente si parece tener datos (no encabezados)
            // Solo saltamos si la columna 2 (cantidad) no es un número.
            let rawQty = row[1] ? row[1].toString().trim() : "";
            let qty = parseInt(rawQty);

            // Si la fila está vacía o el nombre es nulo, saltar
            if (!row[0]) {
                rowsSkipped++;
                return;
            }

            // Si es la primera fila y el valor no es un número, es un encabezado (saltar)
            if (index === 0 && isNaN(qty)) return;

            if (!isNaN(qty) && qty > 0) {
                let name = String(row[0]).replace(/"/g, '').replace(/ATT-CONSIGN|ATT/g, '').trim();
                let brand = name.split(' ')[0] || 'VARIOS';
                
                grandTotal += qty;
                if (!brands[brand]) brands[brand] = [];
                brands[brand].push({ name, qty });
            } else {
                // CORRECCIÓN 2: Mostrar aviso si hay equipos con cantidad 0 o error
                if (row[0] && (isNaN(qty) || qty === 0)) {
                    debug.style.display = 'block';
                    debug.innerHTML += `⚠️ Fila ${index + 1} ignorada: "${row[0]}" tiene cantidad ${row[1]}<br>`;
                }
            }
        });

        document.getElementById('totalGlobal').innerText = `TOTAL: ${grandTotal} UNIDADES`;

        Object.keys(brands).sort().forEach(b => {
            const totalMarca = brands[b].reduce((s, i) => s + i.qty, 0);
            const block = document.createElement('div');
            block.className = 'brand-block';
            block.innerHTML = `
                <div class="brand-header"><span>${b}</span><span>Σ ${totalMarca}</span></div>
                <table>${brands[b].map(i => `<tr><td>${i.name}</td><td class="qty">${i.qty}</td></tr>`).join('')}</table>
            `;
            output.appendChild(block);
        });
    }

    fetchData();
    setInterval(fetchData, 60000);
</script>

</body>
</html>
