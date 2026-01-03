
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Reporte Ejecutivo de Inventario</title>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;800&display=swap');

        :root {
            --primary: #000000;
            --accent: #2563eb;
            --gray-light: #f3f4f6;
            --text-main: #1f2937;
        }

        body { 
            font-family: 'Inter', sans-serif; 
            background: #fdfdfd; margin: 0; padding: 20px; color: var(--text-main);
        }

        /* --- UI DE CONTROL (NO SE IMPRIME) --- */
        .no-print-panel {
            max-width: 900px; margin: 0 auto 30px auto;
            background: white; padding: 20px; border-radius: 12px;
            box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1);
            text-align: center; border: 1px solid #e5e7eb;
        }
        .btn { 
            padding: 12px 24px; border: none; border-radius: 8px; 
            cursor: pointer; font-weight: 600; font-size: 14px;
            transition: all 0.2s; text-transform: uppercase;
        }
        .btn-upload { background: #4f46e5; color: white; margin-right: 12px; }
        .btn-print { background: #10b981; color: white; }
        .btn:hover { opacity: 0.9; transform: translateY(-1px); }

        /* --- DISEÑO DEL REPORTE --- */
        .report-container { max-width: 1000px; margin: 0 auto; }

        .report-header {
            display: flex; justify-content: space-between; align-items: flex-end;
            border-bottom: 2px solid var(--primary);
            padding-bottom: 10px; margin-bottom: 20px;
        }
        .report-header h1 { margin: 0; font-size: 26px; font-weight: 800; letter-spacing: -1px; }
        .meta-info { text-align: right; font-size: 12px; line-height: 1.4; color: #4b5563; }

        #inventoryOutput {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 20px;
        }

        .brand-block {
            break-inside: avoid;
            margin-bottom: 15px;
        }

        .brand-header {
            background: var(--gray-light);
            padding: 5px 10px;
            font-weight: 800; font-size: 12px;
            text-transform: uppercase;
            display: flex; justify-content: space-between;
            border-left: 4px solid var(--primary);
            margin-bottom: 4px;
        }

        table { width: 100%; border-collapse: collapse; }
        tr:nth-child(even) { background-color: #f9fafb; }
        td { 
            padding: 6px 10px; font-size: 11px; 
            border-bottom: 1px solid #edf2f7;
            line-height: 1.2;
        }
        .qty-val { 
            text-align: right; font-weight: 700; width: 35px; 
            color: var(--accent); font-size: 12px;
        }

        #fileInput { display: none; }

        /* --- CONFIGURACIÓN DE IMPRESIÓN MAESTRA --- */
        @media print {
            @page { size: portrait; margin: 0.7cm; }
            body { background: white; padding: 0; }
            .no-print-panel { display: none; }
            
            .report-header { margin-bottom: 15px; border-bottom-width: 3px; }
            .report-header h1 { font-size: 22px; }

            #inventoryOutput {
                display: block;
                column-count: 3; /* Tres columnas perfectas para ahorrar papel */
                column-gap: 20px;
                column-rule: 1px solid #f0f0f0; /* Línea divisoria entre columnas */
            }

            .brand-block { 
                margin-bottom: 12px;
                border: 1px solid #eee; /* Recuadro sutil para organización */
                padding: 2px;
            }
            
            td { padding: 4px 6px; font-size: 10.5px; }
            .brand-header { font-size: 10px; padding: 3px 8px; }
        }
    </style>
</head>
<body>

<div class="no-print-panel">
    <h3 style="margin: 0 0 15px 0; color: #374151;">Generador de Reporte Compacto</h3>
    <input type="file" id="fileInput" accept=".xlsx, .xls">
    <button class="btn btn-upload" onclick="document.getElementById('fileInput').click()">📁 Seleccionar Excel</button>
    <button class="btn btn-print" onclick="window.print()">🖨️ Imprimir Hoja</button>
</div>

<div class="report-container">
    <div class="report-header">
        <div>
            <h1>REPORTE DE INVENTARIO</h1>
            <div id="total-global" style="font-size: 12px; font-weight: 600; color: var(--accent); margin-top: 4px;"></div>
        </div>
        <div class="meta-info">
            <strong id="print-date"></strong><br>
            <span>Documento generado para control de stock</span>
        </div>
    </div>

    <div id="inventoryOutput">
        </div>
</div>

<script>
    // Configurar Fecha
    const opciones = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' };
    document.getElementById('print-date').innerText = new Date().toLocaleDateString('es-ES', opciones).toUpperCase();

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
        let totalGeneral = 0;

        data.forEach((row, index) => {
            if (index === 0 || !row[0]) return;
            
            // Limpiar nombres
            let fullName = String(row[0])
                .replace(/ATT-CONSIGN/g, '')
                .replace(/ATT/g, '')
                .trim();
            
            const brand = fullName.split(' ')[0] || 'VARIOS';
            const qty = parseInt(row[1]) || 0;
            
            totalGeneral += qty;

            if (!brands[brand]) brands[brand] = [];
            brands[brand].push({ name: fullName, qty: qty });
        });

        // Mostrar Total Global
        document.getElementById('total-global').innerText = `EXISTENCIA TOTAL: ${totalGeneral} UNIDADES`;

        // Ordenar y Renderizar
        Object.keys(brands).sort().forEach(brand => {
            const block = document.createElement('div');
            block.className = 'brand-block';
            
            const totalMarca = brands[brand].reduce((s, i) => s + i.qty, 0);
            
            let tableRows = brands[brand].map(item => `
                <tr>
                    <td>${item.name}</td>
                    <td class="qty-val">${item.qty}</td>
                </tr>
            `).join('');

            block.innerHTML = `
                <div class="brand-header">
                    <span>${brand}</span>
                    <span>Σ ${totalMarca}</span>
                </div>
                <table>${tableRows}</table>
            `;
            output.appendChild(block);
        });
    }
</script>

</body>
</html>
