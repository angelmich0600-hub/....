
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Inventario Compacto Pro</title>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <style>
        :root {
            --primary: #2d3436;
            --accent: #0984e3;
            --bg: #f5f6fa;
        }

        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); margin: 0; padding: 20px; }
        .container { max-width: 1000px; margin: 0 auto; }

        header {
            text-align: center;
            padding: 20px;
            background: #000;
            color: white;
            border-radius: 10px;
            margin-bottom: 20px;
        }

        .controls {
            display: flex; gap: 10px; justify-content: center;
            background: white; padding: 15px; border-radius: 10px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1); margin-bottom: 20px;
        }

        .btn { padding: 10px 20px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; }
        .btn-upload { background: #6c5ce7; color: white; }
        .btn-print { background: var(--accent); color: white; }

        .brand-section {
            background: white; border-radius: 8px; padding: 15px;
            margin-bottom: 15px; box-shadow: 0 2px 4px rgba(0,0,0,0.05);
            border-left: 4px solid var(--accent);
        }

        .brand-title { color: var(--accent); font-weight: bold; text-transform: uppercase; border-bottom: 1px solid #eee; margin-bottom: 8px; }

        table { width: 100%; border-collapse: collapse; font-size: 0.9em; }
        th { text-align: left; color: #636e72; font-size: 0.75em; text-transform: uppercase; padding: 5px; }
        td { padding: 6px 5px; border-bottom: 1px solid #f1f2f6; }
        .qty-badge { font-weight: bold; background: #eee; padding: 2px 8px; border-radius: 10px; }

        #fileInput { display: none; }

        /* --- AJUSTES ESPECÍFICOS PARA IMPRESIÓN --- */
        @media print {
            @page { size: auto; margin: 1cm; }
            body { background: white; padding: 0; }
            .controls, header p { display: none; }
            header { padding: 10px; margin-bottom: 10px; }
            header h1 { font-size: 1.5em; margin: 0; color: black !important; }
            
            #inventoryOutput {
                column-count: 2; /* Divide en 2 columnas para ahorrar papel */
                column-gap: 20px;
            }

            .brand-section {
                box-shadow: none; border: 1px solid #ccc;
                margin-bottom: 10px; padding: 8px;
                page-break-inside: avoid; /* Evita que una marca se corte entre dos hojas */
                break-inside: avoid;
            }

            .brand-title { font-size: 1em; margin-bottom: 5px; }
            td { padding: 3px; font-size: 0.8em; } /* Letra más chica */
            th { padding: 3px; }
        }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>REPORTE DE INVENTARIO</h1>
        <p>Carga el archivo Excel para procesar</p>
    </header>

    <div class="controls">
        <input type="file" id="fileInput" accept=".xlsx, .xls">
        <button class="btn btn-upload" onclick="document.getElementById('fileInput').click()">📁 SELECCIONAR EXCEL</button>
        <button class="btn btn-print" onclick="window.print()">🖨️ IMPRIMIR AHORA</button>
    </div>

    <div id="inventoryOutput">
        <p style="text-align:center; color: #999;">Cargue un archivo para ver las tablas...</p>
    </div>
</div>

<script>
    document.getElementById('fileInput').addEventListener('change', function(e) {
        const file = e.target.files[0];
        const reader = new FileReader();
        reader.onload = function(e) {
            const data = new Uint8Array(e.target.result);
            const workbook = XLSX.read(data, {type: 'array'});
            const firstSheet = workbook.Sheets[workbook.SheetNames[0]];
            const jsonData = XLSX.utils.sheet_to_json(firstSheet, {header: 1});
            processData(jsonData);
        };
        reader.readAsArrayBuffer(file);
    });

    function processData(data) {
        const output = document.getElementById('inventoryOutput');
        output.innerHTML = '';
        const brands = {};

        data.forEach((row, index) => {
            if (index === 0 || !row[0]) return;
            const fullName = String(row[0]).replace('ATT-CONSIGN', '').replace('ATT', '').trim();
            const brand = fullName.split(' ')[0];
            const qty = row[1] || 0;
            if (!brands[brand]) brands[brand] = [];
            brands[brand].push({ name: fullName, qty: qty });
        });

        for (let brand in brands) {
            const section = document.createElement('div');
            section.className = 'brand-section';
            let tableRows = brands[brand].map(item => `
                <tr>
                    <td>${item.name}</td>
                    <td style="text-align:right"><span class="qty-badge">${item.qty}</span></td>
                </tr>
            `).join('');

            section.innerHTML = `
                <div class="brand-title">${brand}</div>
                <table>
                    <thead><tr><th>Modelo</th><th style="text-align:right">Cant.</th></tr></thead>
                    <tbody>${tableRows}</tbody>
                </table>
            `;
            output.appendChild(section);
        }
    }
</script>

</body>
</html>
