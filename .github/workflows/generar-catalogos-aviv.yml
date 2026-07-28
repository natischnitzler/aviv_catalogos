// ══════════════════════════════════════════════════════════════════════════════
// AVIV — Generador de catálogos PDF → GitHub Releases
// A diferencia del script de Temponovo, ACÁ NO SE LISTAN CATÁLOGOS A MANO:
// se generan automáticamente uno por cada categoría que exista en Odoo
// (Oro / Anillo, Oro / Aro, Plata / Collar, etc.) — si en Odoo agregan o
// renombran una categoría, en la próxima corrida aparece/desaparece solo.
//
// Uso: GH_TOKEN_RELEASES=xxx node generar-catalogos-aviv.js
//      GH_TOKEN_RELEASES=xxx node generar-catalogos-aviv.js "Anillo"   (filtra por nombre)
// ══════════════════════════════════════════════════════════════════════════════

process.on('uncaughtException', err => {
  if (err.code === 'EPIPE') return;
  console.error('Uncaught:', err.message);
  process.exit(1);
});

const PDFDocument = require('pdfkit');
const axios       = require('axios');
const https       = require('https');
const fs          = require('fs');
const path        = require('path');
const xmlrpc      = require('xmlrpc');
let   sharp;
try { sharp = require('sharp'); } catch(e) { sharp = null; }

const CACHE_PATH     = path.join(__dirname, 'imagenes_cache.json');

const GH_TOKEN      = process.env.GH_TOKEN_RELEASES;
const GH_REPO_OWNER = process.env.GH_REPO_OWNER || 'natischnitzler';   // ⚠️ confirmar owner correcto para Aviv
const GH_REPO_NAME  = process.env.GH_REPO_NAME  || 'aviv-catalogos';   // repo único (código + releases) — debe ser PÚBLICO para que WhatsApp/la web puedan descargar los PDFs sin login
const GH_RELEASE_TAG = 'catalogos-latest';

const ODOO_URL      = process.env.ODOO_URL      || 'https://aviv.odoo.com';
const ODOO_DB       = process.env.ODOO_DB       || 'durlingm-aviv-v17-main-16689505';
const ODOO_USERNAME = process.env.ODOO_USERNAME;
const ODOO_PASSWORD = process.env.ODOO_PASSWORD;

if (!ODOO_USERNAME || !ODOO_PASSWORD) {
  console.error('❌ Faltan credenciales Odoo. Define ODOO_USERNAME y ODOO_PASSWORD como variables de entorno.');
  process.exit(1);
}

if (!GH_TOKEN) {
  console.error('❌ Falta GH_TOKEN_RELEASES. Sin esto no hay dónde subir los catálogos (solo se usa GitHub Releases, no Dropbox).');
  process.exit(1);
}

const HEADER_PATH = path.join(__dirname, 'header_aviv.png');
const HEADER_IMG  = fs.existsSync(HEADER_PATH) ? fs.readFileSync(HEADER_PATH) : null;

// ══════════════════════════════════════════════════════════════════════════════
// CATÁLOGOS — 100% DINÁMICO, NO SE LISTAN A MANO
// ── En Temponovo cada catálogo se definía a mano en un array (CATALOGOS).
//    Para Aviv, en cambio, las categorías de Odoo ya son granulares y limpias
//    (ej. "Oro / Anillo", "Plata / Collar"), así que construimos un catálogo
//    por cada categoría distinta que aparezca en los productos con stock.
// ── Esto se arma en main(), después de traer los productos de Odoo, con
//    construirCatalogosDinamicos(). Si agregan o sacan una categoría en Odoo,
//    el próximo catálogo generado se ajusta solo.
// ══════════════════════════════════════════════════════════════════════════════
function construirCatalogosDinamicos(productos) {
  const categorias = [...new Set(
    productos.map(p => (p.Category || '').trim()).filter(Boolean)
  )].sort((a, b) => a.localeCompare(b, 'es'));

  // Un catálogo por cada categoría exacta de Odoo (ej. "Oro / Aro", "Plata / Aro")
  const porCategoria = categorias.map(cat => ({
    archivo: 'Catalogo_' + cat.replace(/\s*\/\s*/g, '_').replace(/\s+/g, '_') + '.pdf',
    familia: cat,
    titulo: cat.toUpperCase(),   // subtítulo fijo que se muestra en el header del PDF
    orden: 'alfabetico',
  }));

  // Además, un catálogo COMBINADO por tipo de artículo (ej. "Aro"), que junta
  // TODOS los metales que tengan esa subcategoría (Oro / Aro + Plata / Aro +
  // Plata Enchapada / Aro, todos en un solo PDF).
  const items = [...new Set(categorias.map(cat => parsearCategoria(cat).sub))]
    .sort((a, b) => a.localeCompare(b, 'es'));

  const porItem = items.map(sub => ({
    archivo: 'Catalogo_Item_' + sub.replace(/\s*\/\s*/g, '_').replace(/\s+/g, '_') + '.pdf',
    titulo: `${sub.toUpperCase()} — ORO + PLATA`,
    orden: 'alfabetico',
    todosLosProductos: true,
    filtro: p => parsearCategoria((p.Category || '').trim()).sub === sub,
  }));

  return [...porCategoria, ...porItem];
}

// ══════════════════════════════════════════════════════════════════════════════
// HELPERS
// ══════════════════════════════════════════════════════════════════════════════
// ⚠️ Los prefijos de Temponovo (CA-, RL-, QQ-, etc.) no aplican a los SKU de
// Aviv. Si las referencias internas de Aviv usan algún prefijo que se deba
// recortar para mostrar el código "limpio" en el PDF, agrégalo a esta lista.
const PREFIJOS_A_RECORTAR = [];

function limpiarCodigo(code) {
  if (!code) return '';
  for (const p of PREFIJOS_A_RECORTAR) {
    if (code.startsWith(p)) return code.slice(p.length);
  }
  return code;
}

// Separa "Oro / Anillo" -> {grupo:'Oro', sub:'Anillo'}, "Plata Enchapada / Aro"
// -> {grupo:'Plata Enchapada', sub:'Aro'}. El "sub" es lo que usamos para
// unir el mismo tipo de artículo entre distintos metales.
function parsearCategoria(cat) {
  const partes = (cat || '').split('/').map(s => s.trim()).filter(Boolean);
  const grupo = partes[0] || cat;
  const sub   = partes.slice(1).join(' / ') || grupo;
  return { grupo, sub };
}

// Un producto "pertenece" a una familia si su categoría es exactamente esa
// familia, o si es una subcategoría de ella (empieza con "familia / ").
// Esto es lo que permite que las subcategorías nuevas en Odoo se incluyan solas.
function perteneceAFamilia(categoriaProducto, familia) {
  const cat = (categoriaProducto || '').trim();
  return cat === familia || cat.startsWith(familia + ' / ');
}

function productosDeFamilia(todos, familia) {
  return todos.filter(p => perteneceAFamilia(p.Category, familia));
}

// Punto único que usan ambos lugares del script que arman la lista de
// productos de un catálogo — soporta tanto los catálogos "por familia"
// (Oro / Aro) como los combinados "por artículo" (todosLosProductos: true).
function productosDeCatalogo(todos, cat) {
  let prods = cat.todosLosProductos ? todos.slice() : productosDeFamilia(todos, cat.familia);
  if (cat.filtro) prods = prods.filter(cat.filtro);
  return prods;
}

function ordenarProductos(productos, orden) {
  return [...productos].sort((a, b) => {
    if (orden === 'alfabetico')
      return limpiarCodigo(a.Default_code).localeCompare(limpiarCodigo(b.Default_code), 'es');
    const catA = (a.Category||'').trim();
    const catB = (b.Category||'').trim();
    if (catA !== catB) return catA.localeCompare(catB, 'es');
    return limpiarCodigo(a.Default_code).localeCompare(limpiarCodigo(b.Default_code), 'es');
  });
}

// ══════════════════════════════════════════════════════════════════════════════
// ODOO
// ══════════════════════════════════════════════════════════════════════════════
let _uid = null;
let _objectClient = null;

async function getUID() {
  if (_uid) return _uid;
  console.log('🔐 Autenticando con Odoo...');
  const commonClient = xmlrpc.createSecureClient({
    host: new URL(ODOO_URL).hostname, port: 443, path: '/xmlrpc/2/common'
  });
  return new Promise((resolve, reject) => {
    commonClient.methodCall('authenticate',
      [ODOO_DB, ODOO_USERNAME, ODOO_PASSWORD, {}],
      (err, uid) => {
        if (err) return reject(err);
        _uid = uid;
        _objectClient = xmlrpc.createSecureClient({
          host: new URL(ODOO_URL).hostname, port: 443, path: '/xmlrpc/2/object'
        });
        console.log('✅ Odoo UID:', uid);
        resolve(uid);
      }
    );
  });
}

async function odooCall(model, method, args) {
  const uid = await getUID();
  return new Promise((resolve, reject) => {
    _objectClient.methodCall('execute_kw',
      [ODOO_DB, uid, ODOO_PASSWORD, model, method, args],
      (err, res) => err ? reject(err) : resolve(res)
    );
  });
}

// Solo stock y precio — rápido
async function fetchProductos() {
  console.log('📦 Obteniendo productos de Odoo...');
  const raw = await odooCall('product.product', 'search_read', [
    [], ['default_code','name','list_price','qty_available','virtual_available','categ_id',
         'product_tmpl_id','metal_type','rock_type','product_template_attribute_value_ids']
  ]);

  // Nombres de los valores de atributo de variante (ej. "13", "15") en un solo batch,
  // para no hacer una consulta por producto.
  const attrIds = [...new Set(raw.flatMap(p => p.product_template_attribute_value_ids || []))];
  const attrNombres = {};
  if (attrIds.length) {
    try {
      const attrs = await odooCall('product.template.attribute.value', 'read', [attrIds, ['name']]);
      for (const a of attrs) attrNombres[a.id] = a.name;
    } catch(e) { console.log('  ⚠️  Sin medidas/atributos:', e.message); }
  }

  const productos = raw
    .map(p => ({
      Default_code: p.default_code || '',
      Name:         p.name || '',
      Price:        p.list_price || 0,
      Stock:        p.qty_available || 0,
      Incoming:     Math.max(0, (p.virtual_available||0) - (p.qty_available||0)),
      Category:     p.categ_id ? p.categ_id[1].trim() : '',
      TmplId:       p.product_tmpl_id ? p.product_tmpl_id[0] : null,
      Metal:        p.metal_type ? p.metal_type[1] : '',
      Piedra:       p.rock_type ? p.rock_type[1] : '',
      Medida:       (p.product_template_attribute_value_ids || [])
                      .map(id => attrNombres[id]).filter(Boolean).join(', '),
    }))
    // Solo productos con stock > 0 o incoming > 0
    .filter(p => p.Stock > 0 || p.Incoming > 0);
  console.log(`✅ ${productos.length} productos con stock`);
  return productos;
}

async function fetchCaracteristicas() {
  try {
    console.log('📋 Obteniendo descripciones desde Odoo...');
    const raw = await odooCall('product.template', 'search_read', [
      [['description_ecommerce', '!=', false]],
      ['id', 'description_ecommerce']
    ]);
    const map = {};
    for (const t of raw) {
      if (t.description_ecommerce) map[t.id] = t.description_ecommerce;
    }
    console.log(`✅ ${Object.keys(map).length} descripciones obtenidas`);
    return map;
  } catch(e) { console.log('  ⚠️  Sin descripciones:', e.message); return {}; }
}

// Descarga imágenes en lote desde Odoo — solo los códigos que se pidan
async function fetchImagenesEnLote(codes) {
  if (!codes.length) return {};
  const BATCH   = 10;
  const REINTENTOS = 3;
  const imgs  = {};
  for (let i = 0; i < codes.length; i += BATCH) {
    const batch = codes.slice(i, i + BATCH);
    let ok = false;
    for (let intento = 1; intento <= REINTENTOS; intento++) {
      try {
        const raw = await odooCall('product.product', 'search_read', [
          [['default_code', 'in', batch]], ['default_code', 'image_512', 'write_date']
        ]);
        for (const p of raw) {
          if (p.image_512) imgs[p.default_code] = { img: p.image_512, fecha: p.write_date };
        }
        ok = true;
        break;
      } catch(e) {
        if (intento < REINTENTOS) {
          await new Promise(r => setTimeout(r, 2000 * intento));
        }
      }
    }
    process.stdout.write(`\r  📸 ${Math.min(i+BATCH, codes.length)}/${codes.length}`);
  }
  console.log('');
  return imgs;
}

// ══════════════════════════════════════════════════════════════════════════════
// CACHE DE IMÁGENES (archivo local imagenes_cache.json)
// ══════════════════════════════════════════════════════════════════════════════
function cargarCache() {
  try {
    if (fs.existsSync(CACHE_PATH)) {
      const cache = JSON.parse(fs.readFileSync(CACHE_PATH, 'utf8'));
      console.log(`📂 Cache cargado: ${Object.keys(cache).length} imágenes`);
      return cache;
    }
  } catch(e) {}
  console.log('📂 Cache vacío — primera ejecución');
  return {};
}

function guardarCache(cache) {
  fs.writeFileSync(CACHE_PATH, JSON.stringify(cache));
  console.log(`💾 Cache guardado: ${Object.keys(cache).length} imágenes`);
}

// ══════════════════════════════════════════════════════════════════════════════
// GENERADOR PDF
// ══════════════════════════════════════════════════════════════════════════════
async function generarPDF(nombreArchivo, productos, orden, caracteristicas, imgs, tituloCategoria) {
  const MM     = 2.8346;
  const PAGE_W = 210 * MM;
  const PAGE_H = 297 * MM;
  const mg     = 4   * MM;
  const headerH  = (210 * (202/1544)) * MM;   // ratio del banner header_aviv.png (1544x202)
  const footerH  = 10  * MM;
  const COLS     = 3;
  const ROWS     = 4;
  const PER_PG   = 12;
  const cellW    = (PAGE_W - mg*2) / COLS;
  const cellH    = 58  * MM;
  const imgAreaH = 36  * MM;
  const totalH   = ROWS * cellH;
  const subtitleH = 6 * MM;  // espacio para nombre de subcategoría
  const available = PAGE_H - mg*2 - headerH - subtitleH - footerH;
  const vOffset  = subtitleH + (available - totalH) / 2;
  const hasIncoming = productos.some(p => p.Incoming > 0 && p.Stock === 0);

  const doc    = new PDFDocument({ size: 'A4', margin: 0, autoFirstPage: false });
  const chunks = [];
  doc.on('data', c => chunks.push(c));

  const fecha = new Date().toLocaleDateString('es-CL', {day:'2-digit',month:'2-digit',year:'numeric'});
  const hora  = new Date().toLocaleTimeString('es-CL', {hour:'2-digit',minute:'2-digit'});

  function drawHeader(subtitulo) {
    doc.addPage();
    const hdrTop = 3 * MM;
    if (HEADER_IMG) {
      try { doc.image(HEADER_IMG, 0, hdrTop, { width: PAGE_W, height: headerH }); }
      catch(e) { doc.fontSize(10).fillColor('#4b5563').font('Helvetica-Bold').text('AVIV', mg, hdrTop+8); }
    }
    doc.moveTo(0, hdrTop+headerH).lineTo(PAGE_W, hdrTop+headerH)
      .strokeColor('#cccccc').lineWidth(0.5*MM).stroke();
    if (subtitulo) {
      doc.fontSize(9).fillColor('#555555').font('Helvetica-Bold')
        .text(subtitulo.toUpperCase(), mg, hdrTop+headerH+1.5*MM,
          { width: PAGE_W-mg*2, align: 'center' });
    }
  }

  function drawFooter() {
    const fy = PAGE_H - mg - footerH;
    doc.moveTo(mg, fy).lineTo(PAGE_W-mg, fy)
      .strokeColor('#cccccc').lineWidth(0.15*MM).stroke();
    doc.fontSize(8).fillColor('#888888').font('Helvetica')
      .text(`Precios sin IVA  ·  ${fecha}  |  AVIV`,
        mg, fy+1.5*MM, { width: PAGE_W-mg*2, align: 'center' });
    if (hasIncoming) {
      doc.fontSize(7.5).fillColor('#888888').font('Helvetica-Oblique')
        .text('* Productos no disponibles para despacho inmediato',
          mg, fy+5.5*MM, { width: PAGE_W-mg*2, align: 'center' });
    }
  }

  let pageIdx    = 0;
  let currentCat = null;

  if (!productos.length) {
    drawHeader(tituloCategoria || null);
    doc.fontSize(12).fillColor('#888888').font('Helvetica-Oblique')
      .text('No hay productos disponibles en este catálogo', mg, PAGE_H/2,
        { width: PAGE_W - mg*2, align: 'center' });
    drawFooter();
    doc.end();
    return new Promise(resolve => doc.on('end', () => resolve(Buffer.concat(chunks))));
  }

  for (const p of productos) {
    const cat      = (p.Category||'').trim();
    const subcat   = cat.includes('/') ? cat.split('/')[1].trim() : cat;
    const prevSub  = currentCat?.includes('/') ? currentCat.split('/')[1].trim() : currentCat;
    const catChanged = orden === 'categoria' && currentCat !== null && subcat !== prevSub;

    if (pageIdx === 0 || pageIdx >= PER_PG || catChanged) {
      if (pageIdx > 0) drawFooter();
      drawHeader(tituloCategoria || (orden === 'categoria' ? subcat : null));
      pageIdx = 0;
    }
    currentCat = cat;

    const col = pageIdx % COLS;
    const row = Math.floor(pageIdx / COLS);
    const x   = mg + col * cellW;
    const y   = 3*MM + headerH + vOffset + row * cellH;

    doc.rect(x, y, cellW, cellH).fill('#ffffff');

    const cached = imgs[p.Default_code];
    const b64 = cached ? (typeof cached === 'object' ? cached.img : cached) : null;
    if (b64) {
      try {
        let buf = Buffer.from(b64, 'base64');
        // Detectar WEBP (header RIFF) y convertir a JPEG
        if (sharp) {
          try {
            buf = await sharp(buf)
              .resize(300, 300, { fit: 'inside', withoutEnlargement: true })
              .flatten({ background: { r:255, g:255, b:255 } })
              .jpeg({ quality: 60 })
              .toBuffer();
          } catch(se) {}
        }
        doc.rect(x, y, cellW, imgAreaH).fill('#ffffff');
        doc.image(buf, x+1*MM, y+1*MM, {
          fit: [cellW-2*MM, imgAreaH-2*MM],
          align: 'center', valign: 'center'
        });
      } catch(e) { doc.rect(x, y, cellW, imgAreaH).fill('#f5f5f5'); }
    } else {
      doc.rect(x, y, cellW, imgAreaH).fill('#f5f5f5');
    }

    const conStar = p.Incoming > 0 && p.Stock === 0;
    const codigo  = limpiarCodigo(p.Default_code) || '';
    let   infoY   = y + imgAreaH + 1*MM;

    const nombre = (p.Name || '').trim();
    if (nombre) {
      const nombreCorto = nombre.length > 42 ? nombre.slice(0, 41).trim() + '…' : nombre;
      doc.fontSize(7.5).fillColor('#333333').font('Helvetica-Bold')
        .text(nombreCorto, x+1*MM, infoY, { width: cellW-2*MM, align: 'center', lineBreak: false });
      infoY += 3.3*MM;
    }

    doc.fontSize(9).fillColor('#000000').font('Helvetica-Bold')
      .text(codigo + (conStar ? ' *' : ''), x, infoY,
        { width: cellW, align: 'center', lineBreak: false });
    infoY += 3.5*MM;

    doc.fontSize(8.5).fillColor('#000000').font('Helvetica')
      .text(`$${Math.round(p.Price||0).toLocaleString('es-CL')} + IVA`,
        x, infoY, { width: cellW, align: 'center', lineBreak: false });
    infoY += 3.5*MM;

    const metalPiedra = [p.Metal, p.Piedra].filter(Boolean).join('  ·  ');
    if (metalPiedra) {
      doc.fontSize(7).fillColor('#666666').font('Helvetica')
        .text(metalPiedra, x+1*MM, infoY, { width: cellW-2*MM, align: 'center', lineBreak: false });
      infoY += 3*MM;
    }

    if (p.Medida) {
      doc.fontSize(7).fillColor('#666666').font('Helvetica')
        .text(`Medida: ${p.Medida}`, x+1*MM, infoY, { width: cellW-2*MM, align: 'center', lineBreak: false });
    }

    pageIdx++;
  }

  if (pageIdx > 0) drawFooter();
  doc.end();
  return new Promise(resolve => doc.on('end', () => resolve(Buffer.concat(chunks))));
}


// ══════════════════════════════════════════════════════════════════════════════
// GENERADOR HTML — Consulta de stock para celular
// ══════════════════════════════════════════════════════════════════════════════
function generarHTML(todos, fecha, hora) {
  const grupos = {};
  for (const p of todos) {
    const pts   = p.Category.split('/');
    const padre = pts[0].trim();
    const hijo  = pts[1] ? pts[1].trim() : padre;
    if (!grupos[padre]) grupos[padre] = {};
    if (!grupos[padre][hijo]) grupos[padre][hijo] = [];
    grupos[padre][hijo].push(p);
  }
  const padres = Object.keys(grupos).sort((a,b) => a.localeCompare(b,'es'));

  function limpiarCod(c) {
    for (const p of ['CA-CA-','CA-','RL-','QQ-','CC-','ES-','PI-','LI-','CO-','CS-','ZI-'])
      if (c.startsWith(p)) return c.slice(p.length);
    return c;
  }
  function stockStyle(s) {
    const color = s < 10 ? '#c0392b' : s <= 20 ? '#e67e22' : '#4b5563';
    return 'font-size:13px;font-weight:500;min-width:36px;text-align:right;color:' + color;
  }
  function stockLabel(s) { return s > 100 ? '100+' : String(s); }

  let filas = '';
  for (const padre of padres) {
    const hijos = Object.keys(grupos[padre]).sort((a,b) => a.localeCompare(b,'es'));
    filas += `<div class="padre" data-padre="${padre}"><div class="padre-label">${padre}</div><div class="padre-body">`;
    for (const hijo of hijos) {
      const prods = grupos[padre][hijo].sort((a,b) =>
        limpiarCod(a.Default_code||'').localeCompare(limpiarCod(b.Default_code||''),'es'));
      filas += `<div class="hijo-label" data-hijo="${hijo}">${hijo}</div>`;
      for (const p of prods) {
        const cod = limpiarCod(p.Default_code || '');
        filas += `<div class="fila" data-cod="${cod.toLowerCase()}">`;
        filas += `<span class="codigo">${cod}</span>`;
        filas += `<span style="${stockStyle(p.Stock)}">${stockLabel(p.Stock)}</span>`;
        filas += `</div>`;
      }
    }
    filas += `</div></div>`;
  }

  const totalProductos = todos.length;
  const catButtons = padres.map(c => `<span class="cat-btn" data-cat="${c}">${c}</span>`).join('');
  const subDiv = `<div id="subcats"></div>`;

  return `<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="apple-mobile-web-app-capable" content="yes">
<title>Stock Aviv</title>
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;background:#f5f5f5;color:#111}
.header{background:#4b5563;color:#fff;padding:12px 16px;position:sticky;top:0;z-index:10}
.header small{font-size:11px;opacity:.7;display:block;margin-bottom:2px}
.header h1{font-size:17px;font-weight:500}
.search-wrap{padding:10px 12px;background:#fff;border-bottom:1px solid #e5e5e5;position:sticky;top:52px;z-index:9}
.search{display:flex;align-items:center;gap:8px;background:#f5f5f5;border-radius:8px;padding:7px 10px;border:1px solid #e0e0e0}
.search svg{flex-shrink:0;color:#999}
.search input{border:none;background:none;outline:none;font-size:13px;width:100%;color:#111}
.search input::placeholder{color:#aaa}
#cats{display:flex;gap:6px;overflow-x:auto;padding:8px 12px;background:#fff;border-bottom:1px solid #e5e5e5;position:sticky;top:104px;z-index:8;scrollbar-width:none}
#cats::-webkit-scrollbar{display:none}
#subcats{display:none;gap:6px;overflow-x:auto;padding:6px 12px;background:#f2f3f5;border-bottom:1px solid #d8dbe0;position:sticky;top:148px;z-index:7;scrollbar-width:none}
#subcats::-webkit-scrollbar{display:none}
.cat-btn{font-size:11px;padding:3px 10px;border-radius:20px;border:1px solid #ddd;background:#f5f5f5;color:#555;white-space:nowrap;cursor:pointer;flex-shrink:0}
.cat-btn.active{background:#4b5563;color:#fff;border-color:#4b5563}
.sub-btn{font-size:11px;padding:3px 10px;border-radius:20px;border:1px solid #c4c9d0;background:#fff;color:#4b5563;white-space:nowrap;cursor:pointer;flex-shrink:0}
.sub-btn.active{background:#4b5563;color:#fff;border-color:#4b5563}
.content{padding:8px 12px 24px}
.padre-label{font-size:11px;font-weight:500;color:#888;text-transform:uppercase;letter-spacing:.5px;padding:10px 0 4px}
.padre-body{margin-left:8px;border-left:2px solid #e0e0e0;padding-left:10px}
.hijo-label{font-size:11px;font-weight:500;color:#4b5563;padding:6px 0 3px}
.fila{display:flex;align-items:center;justify-content:space-between;padding:4px 0;border-bottom:1px solid #f0f0f0}
.fila:last-child{border-bottom:none}
.codigo{font-size:13px;font-weight:500;color:#111}
.stock-num{font-size:13px!important;font-weight:500;min-width:36px;text-align:right}
.s-red{color:#c0392b}
.s-yellow{color:#e67e22}
.s-green{color:#4b5563}
.footer{display:flex;align-items:center;justify-content:center;gap:12px;padding:10px 12px;font-size:11px;color:#888;border-top:1px solid #eee;background:#fff}
.hidden{display:none!important}
#no-results{text-align:center;padding:32px 16px;color:#aaa;font-size:14px}
</style>
</head>
<body>
<div class="header">
  <small>AVIV</small>
  <h1>Consulta de stock</h1>
</div>
<div class="search-wrap">
  <div class="search">
    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.35-4.35"/></svg>
    <input type="search" id="buscador" placeholder="Buscar por código..." autocomplete="off" autocorrect="off" autocapitalize="off">
  </div>
</div>
<div id="cats">
  <span class="cat-btn active" data-cat="todos">Todos</span>
  ${catButtons}
</div>
<div id="subcats"></div>
<div class="content" id="content">
  ${filas}
  <div class="hidden" id="no-results">Sin resultados</div>
</div>
<div class="footer">
  <span><span style="color:#c0392b">●</span> &lt;10</span>
  <span><span style="color:#e67e22">●</span> 10-20</span>
  <span><span style="color:#4b5563">●</span> &gt;20</span>
  <span style="color:#aaa">· ${fecha} ${hora} · ${totalProductos} productos</span>
</div>
<script>
let catActiva='todos', subActiva='todos';
const subcatsEl=document.getElementById('subcats');
const grupos=${JSON.stringify(Object.fromEntries(padres.map(p=>[p,Object.keys(grupos[p])])))} ;

document.getElementById('cats').addEventListener('click',e=>{
  const btn=e.target.closest('.cat-btn');if(!btn)return;
  document.querySelectorAll('.cat-btn').forEach(b=>b.classList.remove('active'));
  btn.classList.add('active');
  catActiva=btn.dataset.cat;
  subActiva='todos';
  renderSubcats();
  filtrar();
});

function renderSubcats(){
  subcatsEl.innerHTML='';
  if(catActiva==='todos'){subcatsEl.style.display='none';return;}
  const hijos=(grupos[catActiva]||[]).sort((a,b)=>a.localeCompare(b,'es'));
  if(hijos.length<=1){subcatsEl.style.display='none';return;}
  subcatsEl.style.display='flex';
  const t=document.createElement('span');
  t.className='sub-btn active';t.textContent='Todos';t.dataset.sub='todos';
  subcatsEl.appendChild(t);
  hijos.forEach(h=>{
    const b=document.createElement('span');
    b.className='sub-btn';b.textContent=h;b.dataset.sub=h;
    subcatsEl.appendChild(b);
  });
  subcatsEl.onclick=e=>{
    const btn=e.target.closest('.sub-btn');if(!btn)return;
    document.querySelectorAll('.sub-btn').forEach(b=>b.classList.remove('active'));
    btn.classList.add('active');
    subActiva=btn.dataset.sub;
    filtrar();
  };
}

document.getElementById('buscador').addEventListener('input',filtrar);

function filtrar(){
  const q=document.getElementById('buscador').value.toLowerCase().trim();
  let visible=0;
  document.querySelectorAll('.padre').forEach(padreEl=>{
    const matchCat=catActiva==='todos'||padreEl.dataset.padre===catActiva;
    if(!matchCat){padreEl.classList.add('hidden');return;}
    padreEl.classList.remove('hidden');
    let padreVis=false;
    padreEl.querySelectorAll('.hijo-label').forEach(hl=>{
      const matchSub=subActiva==='todos'||hl.dataset.hijo===subActiva;
      if(!matchSub){
        hl.classList.add('hidden');
        let el=hl.nextElementSibling;
        while(el&&el.classList.contains('fila')){el.classList.add('hidden');el=el.nextElementSibling;}
        return;
      }
      let hijoVis=false;
      let el=hl.nextElementSibling;
      while(el&&el.classList.contains('fila')){
        const match=!q||el.dataset.cod.includes(q);
        el.classList.toggle('hidden',!match);
        if(match){hijoVis=true;padreVis=true;visible++;}
        el=el.nextElementSibling;
      }
      hl.classList.toggle('hidden',!hijoVis);
    });
    padreEl.classList.toggle('hidden',!padreVis);
  });
  document.getElementById('no-results').classList.toggle('hidden',visible>0||(!q&&catActiva==='todos'));
}
<\/script>
</body>
</html>`;
}

// ══════════════════════════════════════════════════════════════════════════════
// COMPRESION PDF (solo para GitHub/WhatsApp)
// ══════════════════════════════════════════════════════════════════════════════
const { execSync } = require('child_process');
const os = require('os');

function comprimirPDF(buffer) {
  const tmpIn  = path.join(os.tmpdir(), `in_${Date.now()}.pdf`);
  const tmpOut = path.join(os.tmpdir(), `out_${Date.now()}.pdf`);
  try {
    fs.writeFileSync(tmpIn, buffer);
    execSync(
      `gs -sDEVICE=pdfwrite -dCompatibilityLevel=1.4 -dPDFSETTINGS=/ebook -dNOPAUSE -dQUIET -dBATCH -sOutputFile=${tmpOut} ${tmpIn}`,
      { timeout: 180000 }
    );
    const compressed = fs.readFileSync(tmpOut);
    const orig = (buffer.length/1024/1024).toFixed(1);
    const comp = (compressed.length/1024/1024).toFixed(1);
    console.log(`  🗜  ${orig}MB → ${comp}MB`);
    return compressed;
  } catch(e) {
    console.log(`  ⚠️  Compresion fallida: ${e.message}`);
    return buffer;
  } finally {
    try { fs.unlinkSync(tmpIn); } catch(e) {}
    try { fs.unlinkSync(tmpOut); } catch(e) {}
  }
}

// ══════════════════════════════════════════════════════════════════════════════
// GITHUB RELEASES
// ══════════════════════════════════════════════════════════════════════════════
async function githubReleaseId() {
  try {
    const res = await axios.get(
      `https://api.github.com/repos/${GH_REPO_OWNER}/${GH_REPO_NAME}/releases/tags/${GH_RELEASE_TAG}`,
      { headers: { Authorization: `Bearer ${GH_TOKEN}`, Accept: 'application/vnd.github+json' } }
    );
    return res.data.id;
  } catch(e) {
    if (e.response?.status === 404) {
      const res = await axios.post(
        `https://api.github.com/repos/${GH_REPO_OWNER}/${GH_REPO_NAME}/releases`,
        { tag_name: GH_RELEASE_TAG, name: 'Catálogos Aviv', body: 'PDFs generados automáticamente', draft: false, prerelease: false },
        { headers: { Authorization: `Bearer ${GH_TOKEN}`, Accept: 'application/vnd.github+json' } }
      );
      return res.data.id;
    }
    throw e;
  }
}

async function subirAGithub(buffer, nombreArchivo, releaseId) {
  const nombreLimpio = nombreArchivo.replace(/\s+/g, '_');
  nombreArchivo = nombreLimpio;

  // Borrar asset anterior si existe
  try {
    const assets = await axios.get(
      `https://api.github.com/repos/${GH_REPO_OWNER}/${GH_REPO_NAME}/releases/${releaseId}/assets`,
      { headers: { Authorization: `Bearer ${GH_TOKEN}`, Accept: 'application/vnd.github+json' } }
    );
    const existing = assets.data.find(a => a.name === nombreArchivo);
    if (existing) {
      await axios.delete(
        `https://api.github.com/repos/${GH_REPO_OWNER}/${GH_REPO_NAME}/releases/assets/${existing.id}`,
        { headers: { Authorization: `Bearer ${GH_TOKEN}`, Accept: 'application/vnd.github+json' } }
      );
    }
  } catch(e) {}

  // Subir con fetch nativo (más estable para buffers grandes)
  const { default: nodeFetch } = await import('node-fetch');
  const contentType = nombreArchivo.endsWith('.json') ? 'application/json' : 'application/octet-stream';
  const uploadUrl = `https://uploads.github.com/repos/${GH_REPO_OWNER}/${GH_REPO_NAME}/releases/${releaseId}/assets?name=${encodeURIComponent(nombreArchivo)}`;

  const res = await nodeFetch(uploadUrl, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${GH_TOKEN}`,
      Accept: 'application/vnd.github+json',
      'Content-Type': contentType,
      'Content-Length': String(buffer.length),
    },
    body: buffer,
  });

  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(err.message || `HTTP ${res.status}`);
  }
  const data = await res.json();
  return data.browser_download_url;
}

// ══════════════════════════════════════════════════════════════════════════════
// MAIN
// ══════════════════════════════════════════════════════════════════════════════
async function main() {
  const filtroArg = process.argv[2];

  // 1. Obtener stock y características en paralelo
  console.log('📦 Obteniendo datos de Odoo...');
  const [todos, caracteristicas] = await Promise.all([
    fetchProductos(),
    fetchCaracteristicas()
  ]);
  console.log(`✅ ${todos.length} productos con stock | ${Object.keys(caracteristicas).length} con características\n`);

  // 2. Determinar qué catálogos generar — construidos dinámicamente desde
  //    las categorías reales que traen los productos de Odoo (sin lista a mano)
  const CATALOGOS = construirCatalogosDinamicos(todos);
  console.log(`🗂  ${CATALOGOS.length} categorías detectadas en Odoo`);
  const catalogos = filtroArg
    ? CATALOGOS.filter(c => c.archivo.toLowerCase().includes(filtroArg.toLowerCase()))
    : CATALOGOS;

  if (!catalogos.length) {
    console.error(`❌ No se encontró catálogo con: ${filtroArg}`);
    process.exit(1);
  }

  // 3. Determinar todos los códigos necesarios para esta corrida
  const codigosNecesarios = new Set();
  for (const cat of catalogos) {
    let prods = productosDeCatalogo(todos, cat);
    prods.forEach(p => { if (p.Default_code) codigosNecesarios.add(p.Default_code); });
  }
  console.log(`🖼  Productos que necesitan imagen: ${codigosNecesarios.size}`);

  // 4. Cargar cache y descargar solo las que faltan
  const cache = cargarCache();
  // Pedir write_date de todos los productos necesarios para detectar cambios
  console.log('🔍 Verificando cambios de imágenes...');
  const fechasRaw = await odooCall('product.product', 'search_read', [
    [['default_code', 'in', [...codigosNecesarios]]], ['default_code', 'write_date']
  ]);
  const fechasOdoo = {};
  for (const p of fechasRaw) fechasOdoo[p.default_code] = p.write_date;

  const sinCache = [...codigosNecesarios].filter(c => {
    if (!cache[c]) return true; // no está en cache
    const cached = cache[c];
    // Si el cache tiene formato nuevo con fecha, comparar
    if (cached && typeof cached === 'object' && cached.fecha) {
      return cached.fecha !== fechasOdoo[c]; // cambió → re-descargar
    }
    return false; // cache viejo sin fecha → mantener
  });
  const noModificadas = codigosNecesarios.size - sinCache.length;
  if (noModificadas > 0) console.log(`  ✅ ${noModificadas} imágenes sin cambios`);

  if (sinCache.length > 0) {
    console.log(`  ⬇️  Descargando ${sinCache.length} imágenes nuevas...`);
    const nuevas = await fetchImagenesEnLote(sinCache);
    Object.assign(cache, nuevas);
    guardarCache(cache);
    console.log(`  ✅ ${Object.keys(nuevas).length} imágenes descargadas`);
  } else {
    console.log('  ✅ Todas las imágenes en cache\n');
  }

  // 5. Generar y subir PDFs
  const resultados = { ok: [], error: [] };
  const links = {};

  let releaseId = null;
  if (GH_TOKEN) {
    try {
      releaseId = await githubReleaseId();
      console.log(`🐙 GitHub Release ID: ${releaseId}`);
    } catch(e) {
      console.log(`⚠️  No se pudo conectar a GitHub Releases: ${e.message}`);
    }
  }

  for (const cat of catalogos) {
    console.log(`\n📄 ${cat.archivo}`);

    let prods = productosDeCatalogo(todos, cat);

    if (!prods.length) {
      console.log('  ⚠️  Sin productos — generando PDF vacío');
    } else {
      prods = ordenarProductos(prods, cat.orden);
    }
    console.log(`  📊 ${prods.length} productos, orden: ${cat.orden}`);

    try {
      const buffer = await generarPDF(cat.archivo, prods, cat.orden, caracteristicas, cache, cat.titulo);
      console.log(`  ✅ PDF: ${(buffer.length/1024).toFixed(0)} KB`);

      // GitHub Releases — comprimir si pesa mas de 15MB
      if (GH_TOKEN && releaseId) {
        const mb = buffer.length / 1024 / 1024;
        const tieneEspacios = cat.archivo.includes(' ');
        if (mb > 50) {
          console.log(`  ⏭  GitHub: saltando (${mb.toFixed(0)}MB > 50MB)`);
        } else if (tieneEspacios) {
          console.log(`  ⏭  GitHub: saltando (nombre con espacios)`);
        } else {
          try {
            const bufferGH = mb > 15 ? comprimirPDF(buffer) : buffer;
            const url = await subirAGithub(bufferGH, cat.archivo, releaseId);
            links[cat.archivo] = url;
            console.log(`  🐙 GitHub: ${url}`);
          } catch(ge) {
            console.log(`  ⚠️  GitHub: ${ge.message}`);
          }
        }
      }

      resultados.ok.push(cat.archivo);
    } catch(err) {
      console.error(`  ❌ ${err.response?.data?.error_summary || err.message}`);
      resultados.error.push({ archivo: cat.archivo, error: err.message });
    }
  }

  // 6. Generar y subir HTML de consulta de stock
  if (!filtroArg) {
    console.log('\n📱 Generando consulta de stock HTML...');
    try {
      const fecha = new Date().toLocaleDateString('es-CL', {day:'2-digit',month:'2-digit',year:'numeric'});
      const hora  = new Date().toLocaleTimeString('es-CL', {hour:'2-digit',minute:'2-digit'});
      const html  = generarHTML(todos, fecha, hora);
      if (GH_TOKEN && releaseId) {
        const url = await subirAGithub(Buffer.from(html, 'utf8'), 'Stock_Aviv.html', releaseId);
        console.log(`  🐙 Stock_Aviv.html subido: ${url}`);
      } else {
        console.log('  ⏭  Sin GitHub Release disponible, se omite Stock_Aviv.html');
      }
    } catch(err) {
      console.error('  ❌ Error generando HTML:', err.message);
    }
  }

  // Guardar links en JSON para el bot de WhatsApp
  const linksPath = require('path').join(__dirname, 'catalogos_links.json');
  fs.writeFileSync(linksPath, JSON.stringify(links, null, 2));
  console.log(`\n📋 Links guardados en catalogos_links.json (${Object.keys(links).length} catálogos)`);

  // Subir catalogos_links.json al release de GitHub
  if (GH_TOKEN && releaseId) {
    try {
      const jsonBuffer = fs.readFileSync(linksPath);
      const jsonUrl = await subirAGithub(jsonBuffer, 'catalogos_links.json', releaseId);
      console.log(`  🐙 JSON subido: ${jsonUrl}`);
    } catch(e) {
      console.log(`  ⚠️  No se pudo subir el JSON: ${e.message}`);
    }
  }

  console.log('\n══════════════════════════════════════════════');
  console.log(`✅ OK: ${resultados.ok.length} | ❌ Errores: ${resultados.error.length}`);
  resultados.error.forEach(e => console.log(`   ${e.archivo}: ${e.error}`));
  console.log('══════════════════════════════════════════════');
}

main().catch(err => { console.error('❌ Fatal:', err.message); process.exit(1); });
