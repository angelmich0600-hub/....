
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Inventario Pro Estético</title>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <style>
        :root {
            --primary: #2d3436;
            --accent: #0984e3;
            --bg: #f0f2f5;
            --border: #dcdde1;
        }

        body { 
            font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; 
            background: var(--bg); 
            margin: 0; 
            padding: 20px; 
            color: #2f3640;
        }

        .container { max-width: 900px; margin: 0 auto; }

        header {
            text-align: center;
            padding: 25px;
            background: linear-gradient(135deg, #2d3436 0%, #000000 100%);
            color: white;
            border-radius: 12px;
            margin-bottom: 25px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.1);
        }

        header h1 { margin: 0; font-size: 1.8em; letter-spacing: 1px; }
        header p { margin: 5px 0 0; opacity: 0.8; font-size: 0.9em; }

        .controls {
            display: flex; gap: 15px; justify-content: center;
            background: white; padding: 15px; border-radius: 12px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.05); margin-bottom: 25px;
        }

        .btn { 
            padding: 12px 24px; border: none; border-radius: 8px; 
            cursor: pointer; font-weight: 600; transition: all 0.3s ease;
            text-transform: uppercase; font-size: 0.85em;
        }

        .btn-upload { background: #6c5ce7; color: white; }
        .btn-upload:hover { background: #5b4cc4; transform: translateY(-1px); }
        .btn-print { background: var(--accent); color: white; }
        .btn-print:hover { background: #0873c4; transform: translateY(-1px); }

        /* Contenedor de marcas */
        #inventoryOutput {
            display: grid;
            grid-template-columns: 1fr 1fr; /* 2 columnas para PC */
            gap: 15px;
        }

        .brand-section {
            background: white; 
            border-radius: 10px; 
            padding: 12px;
            border: 1px solid var(--border);
            break-inside: avoid;
            display: flex;
            flex-direction: column;
        }

        .brand-title { 
            color: var(--accent); 
            font-weight: 800; 
            font-size: 1em;
            text-transform: uppercase; 
            padding-bottom: 8px;
            margin-bottom: 8px;
            border-bottom: 2px solid var(--bg);
            display: flex;
            justify-content: space-between;
        }

        table { width: 100%; border-collapse: collapse; }
        td { 
            padding: 6px 4px; 
            font-size: 0.85em; 
            border-bottom: 1px solid #f8f9fa;
        }
        
        .item-name { color: #353b48; }
        .qty-cell { 
            text-align: right; 
            font-weight: 700; 
            color: #2f3640;
            width: 40px;
        }

        #fileInput { display: none; }

        /* --- AJUSTES DE IMPRESIÓN --- */
        @media print {
            @page { size: portrait; margin: 1cm; }
            body { background: white; padding: 0; }
            .container { max-width: 100%; }
            .controls, header p { display: none !important; }
            
            header { 
                padding: 10px; margin-bottom: 15px; 
                background: white !important; color: black !important; 
                border-bottom: 3px solid black; border-radius: 0;
                box-shadow: none;
            }

            #inventoryOutput {
                display: block;
                column-count: 2; /* Dos columnas estéticas */
                column-gap: 20px;
            }

            .brand-section {
                border: 1px solid #eee;
                margin-bottom: 12px;
                page-break-inside: avoid;
                box-shadow: none;
            }

            td { font-size: 11px; }
            .brand-title { font-size: 12px; }
        }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>INVENTARIO DE PRODUCTOS</h1>
        <p>Reporte Organizado por Marcas</p>
    </header>

    <div class="controls">
        <input type="file" id="fileInput" accept=".xlsx, .xls">
        <button class="btn btn-upload" onclick="document.getElementById('fileInput').click()">📁 SELECCIONAR EXCEL</button>
        <button class="btn btn-print" onclick="window.print()">🖨️ IMPRIMIR REPORTE</button>
    </div>

    <div id="inventoryOutput">
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
            
            let fullName = String(row[0])
                .replace(/ATT-CONSIGN/g, '')
                .replace(/ATT/g, '')
                .trim();
            
            const brand = fullName.split(' ')[0] || 'VARIOS';
            const qty = row[1] || 0;
            
            if (!brands[brand]) brands[brand] = [];
            brands[brand].push({ name: fullName, qty: qty });
        });

        const sortedBrands = Object.keys(brands).sort();

        sortedBrands.forEach(brand => {
            const section = document.createElement('div');
            section.className = 'brand-section';
            
            // Calculamos total por marca para el encabezado
            const totalQty = brands[brand].reduce((sum, item) => sum + Number(item.qty), 0);
            
            let tableRows = brands[brand].map(item => `
                <tr>
                    <td class="item-name">${item.name}</td>
                    <td class="qty-cell">${item.qty}</td>
                </tr>
            `).join('');

            section.innerHTML = `
                <div class="brand-title">
                    <span>${brand}</span>
                    <span style="font-size: 0.8em; color: #636e72;">Total: ${totalQty}</span>
                </div>
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
