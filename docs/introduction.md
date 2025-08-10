import os
import json
import uuid
import datetime
import shutil
import webbrowser
import PySimpleGUI as sg
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas

# --- Config
DB_FILE = 'sibilante_db.json'
EVIDENCE_DIR = 'evidencias'
PDF_DIR = 'informes'
os.makedirs(EVIDENCE_DIR, exist_ok=True)
os.makedirs(PDF_DIR, exist_ok=True)

# --- Helpers

def load_db():
    if not os.path.exists(DB_FILE):
        return {'users': [], 'cases': []}
    with open(DB_FILE, 'r', encoding='utf-8') as f:
        return json.load(f)


def save_db(db):
    with open(DB_FILE, 'w', encoding='utf-8') as f:
        json.dump(db, f, ensure_ascii=False, indent=2)


def now_iso():
    return datetime.datetime.utcnow().isoformat() + 'Z'


def generate_id():
    return str(uuid.uuid4())

# --- PDF Report

def generate_pdf_report(case):
    filename = f"{PDF_DIR}/informe_{case['id']}.pdf"
    c = canvas.Canvas(filename, pagesize=A4)
    width, height = A4
    y = height - 50
    c.setFont('Helvetica-Bold', 16)
    c.drawString(50, y, f"INFORME SIBILANTE - Caso: {case['titulo']}")
    y -= 30
    c.setFont('Helvetica', 10)
    c.drawString(50, y, f"ID: {case['id']}")
    y -= 15
    c.drawString(50, y, f"Estado: {case.get('estado','abierto')}")
    y -= 15
    c.drawString(50, y, f"Creado por: {case.get('creado_por','-')}  Fecha: {case.get('fecha_creacion','-')}")
    y -= 25

    # Metadatos
    c.setFont('Helvetica-Bold', 12)
    c.drawString(50, y, 'Datos del caso:')
    y -= 18
    c.setFont('Helvetica', 10)
    lines = [f"Descripcion: {case.get('descripcion','-')}", f"Coordenadas: {case.get('lat','-')}, {case.get('lon','-')}"]
    for ln in lines:
        c.drawString(60, y, ln)
        y -= 14
    y -= 8

    # Cronologia
    c.setFont('Helvetica-Bold', 12)
    c.drawString(50, y, 'Cronologia:')
    y -= 18
    c.setFont('Helvetica', 9)
    for entry in case.get('cronologia', []):
        text = f"[{entry['fecha']}] ({entry['etapa']}) {entry['autor']}: {entry['nota']}"
        for chunk in split_text(text, 90):
            c.drawString(60, y, chunk)
            y -= 12
            if y < 80:
                c.showPage()
                y = height - 50
    y -= 8

    # Evidencias
    c.setFont('Helvetica-Bold', 12)
    c.drawString(50, y, 'Evidencias:')
    y -= 18
    c.setFont('Helvetica', 9)
    for ev in case.get('evidencias', []):
        line = f"- {ev['filename']} (guardada: {ev['guardado_en']})"
        c.drawString(60, y, line)
        y -= 12
        if y < 80:
            c.showPage()
            y = height - 50

    c.showPage()
    c.save()
    return filename


def split_text(text, maxchars):
    words = text.split(' ')
    lines = []
    cur = ''
    for w in words:
        if len(cur) + len(w) + 1 <= maxchars:
            cur = cur + ' ' + w if cur else w
        else:
            lines.append(cur)
            cur = w
    if cur: lines.append(cur)
    return lines

# --- Simple auth (local)

def ensure_admin(db):
    if not db['users']:
        db['users'].append({'username':'admin','password':'admin','role':'admin'})
        save_db(db)


def authenticate(db, username, password):
    for u in db['users']:
        if u['username'] == username and u['password'] == password:
            return u
    return None

# --- GUI Helpers

def case_to_row(case):
    return [case['id'], case['titulo'], case.get('estado','abierto'), case.get('creado_por','-'), case.get('fecha_creacion','-')]

# --- Main UI

def main():
    sg.theme('DarkBlue3')
    db = load_db()
    ensure_admin(db)

    # --- Login Window
    login_layout = [
        [sg.Text('SIBILANTE - Inicio de sesi\\u00f3n', font=('Any',14))],
        [sg.Text('Usuario', size=(8,1)), sg.Input(key='-USER-')],
        [sg.Text('Contrase\\u00f1a', size=(8,1)), sg.Input(key='-PASS-', password_char='*')],
        [sg.Button('Entrar'), sg.Button('Salir')]
    ]
    login_win = sg.Window('SIBILANTE - Login', login_layout)
    user = None
    while True:
        ev, vals = login_win.read()
        if ev in (None, 'Salir'):
            login_win.close()
            return
        if ev == 'Entrar':
            u = authenticate(db, vals['-USER-'], vals['-PASS-'])
            if u:
                user = u
                sg.popup('Bienvenido', u['username'])
                login_win.close()
                break
            else:
                sg.popup('Credenciales invalidas')

    # --- Main layout
    case_table_head = ['ID','Titulo','Estado','Creado por','Fecha creacion']
    cases_rows = [case_to_row(c) for c in db.get('cases',[])]

    layout = [
        [sg.Text('SIBILANTE - Panel Operativo', font=('Any',16))],
        [sg.Button('Nuevo Caso'), sg.Button('Ver / Editar'), sg.Button('Generar Informe'), sg.Button('Importar Evidencia'), sg.Button('Usuarios'), sg.Button('Salir')],
        [sg.Table(values=cases_rows, headings=case_table_head, key='-TABLE-', enable_events=True, auto_size_columns=False, col_widths=[36,20,10,12,18], num_rows=10)],
        [sg.Text('Buscar por titulo:'), sg.Input(key='-FILTRO-'), sg.Button('Filtrar')]
    ]

    main_win = sg.Window('SIBILANTE - Operaciones', layout, finalize=True)

    while True:
        ev, vals = main_win.read()
        if ev in (None, 'Salir'):
            break
        if ev == 'Nuevo Caso':
            crear_caso_window(db, user)
            db = load_db()
            main_win['-TABLE-'].update([case_to_row(c) for c in db.get('cases',[])])
        if ev == 'Ver / Editar':
            sel = vals['-TABLE-']
            if not sel:
                sg.popup('Selecciona un caso de la tabla')
            else:
                idx = sel[0]
                case = db['cases'][idx]
                ver_editar_caso(db, case, user)
                db = load_db()
                main_win['-TABLE-'].update([case_to_row(c) for c in db.get('cases',[])])
        if ev == 'Generar Informe':
            sel = vals['-TABLE-']
            if not sel:
                sg.popup('Selecciona un caso')
            else:
                case = db['cases'][sel[0]]
                path = generate_pdf_report(case)
                sg.popup('Informe generado:', path)
                if sg.popup_yes_no('Abrir informe en carpeta?') == 'Yes':
                    webbrowser.open(os.path.abspath(PDF_DIR))
        if ev == 'Importar Evidencia':
            sel = vals['-TABLE-']
            if not sel:
                sg.popup('Selecciona un caso para importar evidencia')
            else:
                case = db['cases'][sel[0]]
                importar_evidencia(case)
                save_db(db)
                sg.popup('Evidencia importada')
        if ev == 'Usuarios':
            gestionar_usuarios(db)
            db = load_db()
            main_win['-TABLE-'].update([case_to_row(c) for c in db.get('cases',[])])
        if ev == 'Filtrar':
            filtro = vals['-FILTRO-'].lower()
            rows = []
            for c in db.get('cases',[]):
                if filtro in c['titulo'].lower():
                    rows.append(case_to_row(c))
            main_win['-TABLE-'].update(rows)

    main_win.close()

# --- Sub-windows

def crear_caso_window(db, user):
    layout = [
        [sg.Text('Nuevo Caso SIBILANTE', font=('Any',14))],
        [sg.Text('Titulo', size=(10,1)), sg.Input(key='-TIT-')],
        [sg.Text('Descripcion', size=(10,1)), sg.Multiline(key='-DESC-', size=(60,6))],
        [sg.Text('Latitud'), sg.Input(key='-LAT-', size=(15,1)), sg.Text('Longitud'), sg.Input(key='-LON-', size=(15,1))],
        [sg.Button('Crear'), sg.Button('Cancelar')]
    ]
    win = sg.Window('Nuevo Caso', layout)
    while True:
        ev, vals = win.read()
        if ev in (None, 'Cancelar'):
            win.close(); return
        if ev == 'Crear':
            caso = {
                'id': generate_id(),
                'titulo': vals['-TIT-'] or 'Sin titulo',
                'descripcion': vals['-DESC-'] or '',
                'lat': vals['-LAT-'] or '',
                'lon': vals['-LON-'] or '',
                'estado': 'abierto',
                'creado_por': user['username'],
                'fecha_creacion': now_iso(),
                'cronologia': [],
                'evidencias': []
            }
            db['cases'].append(caso)
            save_db(db)
            sg.popup('Caso creado')
            win.close(); return


def ver_editar_caso(db, case, user):
    def refresh_evidence_list(case):
        evrows = [[ev['filename'], ev['guardado_en']] for ev in case.get('evidencias',[])]
        return evrows

    layout = [
        [sg.Text(f"Caso: {case['titulo']}", font=('Any',14))],
        [sg.Text('Estado'), sg.Combo(['abierto','en progreso','cerrado'], default_value=case.get('estado','abierto'), key='-EST-'), sg.Button('Guardar Estado')],
        [sg.Text('Descripcion')],
        [sg.Multiline(case.get('descripcion',''), key='-DESC-', size=(70,6))],
        [sg.Frame('Cronologia', [[sg.Listbox(values=[f"[{e['fecha']}] {e['etapa']} - {e['autor']}: {e['nota']}" for e in case.get('cronologia',[])], size=(80,6), key='-CRONO-')],[sg.Input(key='-NOTA-'), sg.Combo(['Seguimiento','Identificacion','Busqueda','Infiltracion','Localizacion','Analisis','Neutralizacion','Tactica','Evaluacion'], key='-ETAPA-'), sg.Button('Agregar Nota')]])],
        [sg.Frame('Evidencias', [[sg.Listbox(values=[ev['filename'] for ev in case.get('evidencias',[])], size=(60,4), key='-EVLIST-'), sg.Button('Abrir Evidencia')], [sg.Button('Importar Archivo')]])],
        [sg.Button('Guardar y Salir')]
    ]
    win = sg.Window('Ver / Editar Caso', layout, finalize=True)
    while True:
        ev, vals = win.read()
        if ev in (None, 'Guardar y Salir'):
            # guardar cambios
            case['descripcion'] = vals['-DESC-']
            case['estado'] = vals['-EST-']
            save_db(db)
            win.close(); return
        if ev == 'Agregar Nota':
            nota = vals['-NOTA-']
            etapa = vals['-ETAPA-']
            if not nota or not etapa:
                sg.popup('Completa etapa y nota')
            else:
                entry = {'fecha': now_iso(), 'etapa': etapa, 'nota': nota, 'autor': user['username']}
                case.setdefault('cronologia', []).append(entry)
                save_db(db)
                win['-CRONO-'].update([f"[{e['fecha']}] {e['etapa']} - {e['autor']}: {e['nota']}" for e in case.get('cronologia',[])])
                sg.popup('Nota agregada')
        if ev == 'Importar Archivo':
            file = sg.popup_get_file('Selecciona archivo', no_window=True)
            if file:
                dest = importar_archivo_a_evidencias(file)
                evmeta = {'filename': os.path.basename(dest), 'original': file, 'guardado_en': dest}
                case.setdefault('evidencias', []).append(evmeta)
                save_db(db)
                win['-EVLIST-'].update([ev['filename'] for ev in case.get('evidencias',[])])
                sg.popup('Archivo importado')
        if ev == 'Abrir Evidencia':
            sel = win['-EVLIST-'].get()
            if sel:
                fname = sel[0]
                for e in case.get('evidencias',[]):
                    if e['filename'] == fname:
                        path = e['guardado_en']
                        if os.path.exists(path):
                            os.startfile(path) if os.name == 'nt' else webbrowser.open(os.path.abspath(path))
                        else:
                            sg.popup('Archivo no encontrado')
