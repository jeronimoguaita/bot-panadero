"""
App web (Streamlit) del Asistente de Panaderia.

- Chat natural: el panadero escribe que pan quiere hacer, cuantos y la
  temperatura ambiente.
- La IA conversa y PIDE los datos que falten, pero TODOS los numeros los
  calcula la calculadora de Python (calculadora_panaderia.py) a traves de
  "herramientas" (function calling). Asi los resultados son siempre exactos.
- Protegida con contrasena. La API key y la contrasena se cargan desde los
  secrets de Streamlit (nunca van en el codigo).

Para correrla:
    streamlit run app.py
"""

import os
import json
import streamlit as st
from openai import OpenAI

import calculadora_panaderia as calc


# ============================================================
# CONFIGURACION
# ============================================================
MODELO_CHAT = "gpt-4o-mini"   # barato y suficiente; la matematica la hace Python
TEMPERATURA = 0.2

st.set_page_config(page_title="Asistente de Panaderia", page_icon="🥖")


# ============================================================
# CONTRASENA DE ACCESO
# ============================================================
def pedir_password():
    """Muestra una pantalla de contrasena. Si no coincide, frena la app."""
    password_correcta = st.secrets.get("APP_PASSWORD", os.environ.get("APP_PASSWORD"))
    if not password_correcta:
        # Si no se configuro contrasena, se permite el acceso (uso local).
        return
    if st.session_state.get("acceso_ok"):
        return
    st.title("🥖 Asistente de Panaderia")
    clave = st.text_input("Contrasena", type="password")
    if st.button("Entrar"):
        if clave == password_correcta:
            st.session_state["acceso_ok"] = True
            st.rerun()
        else:
            st.error("Contrasena incorrecta.")
    st.stop()


def get_client():
    api_key = st.secrets.get("OPENAI_API_KEY", os.environ.get("OPENAI_API_KEY"))
    if not api_key:
        st.error("Falta la OPENAI_API_KEY en los secrets de Streamlit.")
        st.stop()
    return OpenAI(api_key=api_key)


# ============================================================
# HERRAMIENTAS (la IA las llama; los numeros los hace Python)
# ============================================================
HERRAMIENTAS = [
    {
        "type": "function",
        "function": {
            "name": "listar_recetas",
            "description": ("Devuelve los panes que el bot tiene cargados con su receta "
                            "(hidratacion, integral, centeno, sal, semillas, aceite)."),
            "parameters": {"type": "object", "properties": {}},
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calcular_produccion",
            "description": ("CALCULO PRINCIPAL. Dado un pan conocido, la cantidad de piezas y la "
                            "temperatura ambiente, devuelve TODO: ingredientes por pieza y total, "
                            "la masa madre necesaria, la zona termica (ratio, % MM segun temperatura, "
                            "tiempos de bloque/maduracion/retardo). El % de masa madre lo decide la "
                            "temperatura ambiente (no la receta). Usa esta herramienta primero."),
            "parameters": {
                "type": "object",
                "properties": {
                    "pan": {"type": "string", "description": "Nombre del pan (ej. 'pan rustico', 'integral', 'baguette')."},
                    "n_piezas": {"type": "integer", "description": "Cantidad de piezas."},
                    "peso_pieza_g": {"type": "number", "description": "Peso de masa por pieza en gramos. Si no se sabe, omitir y se usa el sugerido de la receta."},
                    "temp_ambiente": {"type": "number", "description": "Temperatura ambiente en grados Celsius."},
                },
                "required": ["pan", "n_piezas", "temp_ambiente"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "clasificar_zona",
            "description": ("Dada la temperatura ambiente (C), devuelve la zona termica "
                            "con el ratio de refresco, el % de masa madre sugerido, y los tiempos "
                            "de fermentacion en bloque, maduracion y retardo."),
            "parameters": {
                "type": "object",
                "properties": {
                    "temp_ambiente": {"type": "number", "description": "Temperatura ambiente en grados Celsius."}
                },
                "required": ["temp_ambiente"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calcular_ingredientes",
            "description": ("Calcula ingredientes por % panadero para un pan que NO este cargado "
                            "(receta a medida). Los % son respecto de la harina total. La masa madre "
                            "ya esta incluida dentro de la harina y la hidratacion (no se suma aparte)."),
            "parameters": {
                "type": "object",
                "properties": {
                    "n_piezas": {"type": "integer", "description": "Cantidad de piezas."},
                    "peso_pieza_g": {"type": "number", "description": "Peso de masa por pieza, en gramos."},
                    "hidratacion_pct": {"type": "number", "description": "Agua como % de la harina (ej. 70)."},
                    "sal_pct": {"type": "number", "description": "Sal como % de la harina (ej. 2)."},
                    "mm_pct": {"type": "number", "description": "Masa madre como % de la harina (segun temperatura)."},
                    "integral_pct": {"type": "number", "description": "Harina integral como % de la harina total (opcional)."},
                    "centeno_pct": {"type": "number", "description": "Harina de centeno como % de la harina total (opcional)."},
                    "semillas_pct": {"type": "number", "description": "Semillas como % de la harina, suman peso extra (opcional)."},
                    "aceite_pct": {"type": "number", "description": "Aceite como % de la harina, suma peso extra (opcional)."},
                },
                "required": ["n_piezas", "peso_pieza_g", "hidratacion_pct", "sal_pct", "mm_pct"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calcular_masa_madre_semilla",
            "description": ("Calcula cuanta masa madre semilla (M0) hay que partir para llegar, "
                            "luego de varios refrescos, a la masa madre final deseada. "
                            "Formula: M0 = Mf / (1 + P_H + P_Ag)^k."),
            "parameters": {
                "type": "object",
                "properties": {
                    "masa_final_g": {"type": "number", "description": "Masa madre final deseada, en gramos (la que pide la receta)."},
                    "p_harina": {"type": "number", "description": "Proporcion de harina del ratio (ej. ratio 1:4:4 -> 4)."},
                    "p_agua": {"type": "number", "description": "Proporcion de agua del ratio (ej. ratio 1:4:4 -> 4)."},
                    "refrescos": {"type": "integer", "description": "Cantidad de refrescos (k)."},
                },
                "required": ["masa_final_g", "p_harina", "p_agua", "refrescos"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "temperatura_agua",
            "description": ("Calcula la temperatura del agua para llegar a la temperatura objetivo de la masa. "
                            "Formula: T_agua = 4*T_obj - (T_harina + T_amb + T_mm + F), con F=4 (amasadora) o F=2 (a mano)."),
            "parameters": {
                "type": "object",
                "properties": {
                    "temp_objetivo": {"type": "number", "description": "Temperatura objetivo de la masa final (C)."},
                    "temp_harina": {"type": "number", "description": "Temperatura de la harina (C)."},
                    "temp_ambiente": {"type": "number", "description": "Temperatura ambiente (C)."},
                    "temp_masamadre": {"type": "number", "description": "Temperatura de la masa madre (C)."},
                    "metodo": {"type": "string", "enum": ["amasadora", "mano"], "description": "Metodo de amasado."},
                },
                "required": ["temp_objetivo", "temp_harina", "temp_ambiente", "temp_masamadre"],
            },
        },
    },
]


def ejecutar_herramienta(nombre, args):
    """Despacha la llamada de la IA a la funcion real de Python."""
    if nombre == "listar_recetas":
        return calc.listar_recetas()
    if nombre == "calcular_produccion":
        return calc.calcular_produccion(
            args["pan"], args["n_piezas"], args.get("peso_pieza_g"), args["temp_ambiente"])
    if nombre == "clasificar_zona":
        return calc.clasificar_zona(args["temp_ambiente"])
    if nombre == "calcular_ingredientes":
        return calc.calcular_ingredientes(
            args["n_piezas"], args["peso_pieza_g"], args["hidratacion_pct"],
            args["sal_pct"], args["mm_pct"], integral_pct=args.get("integral_pct", 0),
            centeno_pct=args.get("centeno_pct", 0), semillas_pct=args.get("semillas_pct", 0),
            aceite_pct=args.get("aceite_pct", 0))
    if nombre == "calcular_masa_madre_semilla":
        return calc.calcular_masa_madre_semilla(
            args["masa_final_g"], args["p_harina"], args["p_agua"], args["refrescos"])
    if nombre == "temperatura_agua":
        return calc.temperatura_agua(
            args["temp_objetivo"], args["temp_harina"], args["temp_ambiente"],
            args["temp_masamadre"], args.get("metodo", "amasadora"))
    return {"error": f"Herramienta desconocida: {nombre}"}


# ============================================================
# SYSTEM PROMPT
# ============================================================
SYSTEM_PROMPT = """
Sos un asistente experto en panaderia artesanal con masa madre. Ayudas al
panadero a planificar su produccion del dia.

PANES CARGADOS: Pan Rustico, Rustico Semillas, Integral, Baguette y Focaccia.
Si te piden uno que no esta (ej. Pan de Molde o Croissant), decilo y ofrece
calcularlo a medida con calcular_ingredientes pidiendo los % (hidratacion, etc.).

FLUJO DE TRABAJO:
- El panadero te dice QUE pan quiere hacer, CUANTAS unidades y la TEMPERATURA
  AMBIENTE. Si falta la temperatura ambiente, pedila (es clave). El peso por
  pieza es opcional: si no lo dice, calcular_produccion usa el sugerido.
- Para el calculo principal usa SIEMPRE calcular_produccion (pan, n_piezas,
  temp_ambiente y, si lo sabe, peso_pieza_g). Te devuelve ingredientes por pieza
  y total, la masa madre necesaria y la zona (ratio, % MM, tiempos).
- IMPORTANTE: el % de masa madre lo decide la TEMPERATURA AMBIENTE, no la receta.
  Mas calor -> menos masa madre (relacion inversa).
- Para saber cuanta masa madre semilla partir usa calcular_masa_madre_semilla,
  pasando como masa_final_g la "masa_madre_necesaria_g" del calculo anterior, y
  como p_harina / p_agua las proporciones del ratio de la zona (ej. ratio 1:4:4
  -> p_harina=4, p_agua=4). Preguntá cuantos refrescos hara.
- Para la temperatura del agua usa temperatura_agua (necesitas temperatura
  objetivo de la masa, de la harina, ambiente, de la masa madre y el metodo de
  amasado: amasadora o a mano).
- Si te preguntan que panes hay, usa listar_recetas.

REGLAS IMPORTANTES:
- NUNCA inventes ni calcules numeros vos mismo: TODOS los numeros tienen que venir
  de las herramientas. Vos solo conversas y explicas el resultado.
- El % de masa madre es un RANGO segun la temperatura; por defecto se usa el punto
  medio. Avisá que se puede ajustar.
- Mostra los resultados de forma clara y ordenada (tablas o listas con unidades:
  g, kg, C, %). Diferencia "por pieza" y "total".
- Se claro y concreto; el panadero trabaja rapido.
"""


# ============================================================
# APP
# ============================================================
def main():
    pedir_password()
    client = get_client()

    st.title("🥖 Asistente de Panaderia")
    st.caption("Decime que pan vas a hacer, cuantos y la temperatura ambiente.")

    if "mensajes" not in st.session_state:
        st.session_state["mensajes"] = [{"role": "system", "content": SYSTEM_PROMPT}]

    # Mostrar historial (saltando el system y los mensajes internos de herramientas)
    for m in st.session_state["mensajes"]:
        if m["role"] in ("user", "assistant") and m.get("content"):
            with st.chat_message(m["role"]):
                st.markdown(m["content"])

    pregunta = st.chat_input("Ej: voy a hacer 20 baguettes de 300 g, hay 24 grados")
    if not pregunta:
        return

    st.session_state["mensajes"].append({"role": "user", "content": pregunta})
    with st.chat_message("user"):
        st.markdown(pregunta)

    with st.chat_message("assistant"):
        with st.spinner("Calculando..."):
            respuesta = responder(client)
        st.markdown(respuesta)


def responder(client) -> str:
    """Ciclo de chat con herramientas: la IA puede pedir calculos varias veces
    antes de dar la respuesta final."""
    for _ in range(6):  # tope de seguridad de iteraciones
        resp = client.chat.completions.create(
            model=MODELO_CHAT,
            messages=st.session_state["mensajes"],
            tools=HERRAMIENTAS,
            temperature=TEMPERATURA,
        )
        msg = resp.choices[0].message

        if not msg.tool_calls:
            st.session_state["mensajes"].append({"role": "assistant", "content": msg.content})
            return msg.content

        # La IA pidio uno o mas calculos: los ejecutamos y le devolvemos el resultado.
        st.session_state["mensajes"].append({
            "role": "assistant",
            "content": msg.content,
            "tool_calls": [tc.model_dump() for tc in msg.tool_calls],
        })
        for tc in msg.tool_calls:
            args = json.loads(tc.function.arguments or "{}")
            resultado = ejecutar_herramienta(tc.function.name, args)
            st.session_state["mensajes"].append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(resultado, ensure_ascii=False),
            })

    return "No pude completar el calculo. Probá reformular el pedido."


if __name__ == "__main__":
    main()
