
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Inventario Ultra Compacto</title>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <style>
        :root {
            --primary: #2d3436;
            --accent: #0984e3;
            --bg: #f5f6fa;
        }

        body { font-family: 'Segoe UI', Arial, sans-serif; background: var(--bg); margin: 0; padding: 10px; }
        .container { max-width: 1100px; margin: 0 auto; }

        header {
            text-align: center;
            padding: 10px;
            background: #000;
            color: white;
            border-radius: 8px;
            margin-bottom: 15px;
        }

        .controls {
            display: flex; gap: 10px; justify-content: center;
            background: white; padding: 10px; border-radius: 8px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1); margin-bottom: 15px;
        }

        .btn { padding: 8px 16px; border: none; border-radius: 5px; cursor: pointer; font-weight: bold; }
        .btn-upload { background: #6c5ce7; color: white; }
        .btn-print { background: var(--accent); color: white; }

        .brand-section {
            background: white; border-radius: 5px; padding: 8px;
            margin-bottom: 8px; box-shadow: 0 1px 3px rgba(0,0,0,0.05);
            border-left: 3px solid var(--accent);
            break-inside: avoid; /* Importante para no cortar marcas */
        }

        .brand-title { 
            color: var(--accent); 
            font-weight: bold; 
            font-size: 0.85em;
            text-transform: uppercase; 
            border-bottom: 1px solid #eee; 
            margin-bottom: 4px; 
        }

        table { width: 100%; border-collapse: collapse; font-size: 0.85em; }
        td { padding: 3px 2px; border-bottom: 1px solid #f1f2f6; }
        .qty-badge { font-weight: bold; color: #000; }
        
        #fileInput { display: none; }

        /* --- AJUSTES DE IMPRESIÓN EXTREMOS --- */
        @media print {
            @page { 
                size: portrait; 
                margin: 0.5cm; /* Márgenes mínimos de impresora */
            }
            body { background: white; padding: 0; }
            .controls, header p, .no-print { display: none !important; }
            header { padding: 5px; margin-bottom: 10px; background: transparent !important; color: black !important; border: 1px solid #eee; }
            header h1 { font-size: 1.2em; margin: 0; }

            #inventoryOutput {
                display: block;
                column-count: 3; /* Dividir en 3 columnas para máximo ahorro */
                column-gap: 10px;
                column-fill: auto;
            }

            .brand-section {
                border: 1px solid #eee;
                margin-bottom: 5px;
                padding: 5px;
                page-break-inside: avoid;
            }

            td { 
                font-size: 10px; /* Tamaño de letra pequeño pero legible */
                line-height: 1.1;
            }
            
            .brand-title { font-size: 11px; margin-bottom: 2px; }
            
            /* Ocultar encabezados de tabla para ganar filas */
            thead { display: none; } 
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
        <p style="text-align:center; color: #999;">Seleccione un archivo para generar la vista compacta...</p>
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
            // Limpieza de nombres
            let fullName = String(row[0])
                .replace(/ATT-CONSIGN/g, '')
                .replace(/ATT/g, '')
                .trim();
            
            const brand = fullName.split(' ')[0] || 'OTROS';
            const qty = row[1] || 0;
            
            if (!brands[brand]) brands[brand] = [];
            brands[brand].push({ name: fullName, qty: qty });
        });

        // Ordenar marcas alfabéticamente
        const sortedBrands = Object.keys(brands).sort();

        sortedBrands.forEach(brand => {
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
                    <tbody>${tableRows}</tbody>
                </table>
            `;
            output.appendChild(section);
        });
    }
</script>

</body>
</html>
