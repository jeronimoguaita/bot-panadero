"""
Calculadora de panaderia (numeros EXACTOS, hechos en Python).

Implementa las formulas escritas a mano del panadero + las recetas de su Excel
(hoja "Calculadora Pan"):

  1) Tabla de ratios de refresco / % de masa madre segun temperatura ambiente.
  2) Recetas de cada pan en % panadero (hidratacion, integral, centeno, sal, etc.).
  3) Calculo de ingredientes con el MISMO modelo que el Excel: la masa madre
     (prefermento) NO se suma aparte; su harina y su agua ya estan contadas
     dentro de la harina total y de la hidratacion.
  4) Formula de la semilla (cuanta masa madre partir): M0 = Mf / (1+P_H+P_Ag)^k
  5) Temperatura del agua: T_agua = 4*T_obj - (T_harina + T_amb + T_mm + F)

El % de masa madre lo decide la TEMPERATURA AMBIENTE (no el valor fijo del Excel).
El chat (LLM) conversa y pide datos, pero TODOS los numeros salen de aca.
"""

from typing import Optional


# ============================================================
# 1) TABLA DE ZONAS SEGUN TEMPERATURA AMBIENTE
#    La proporcion de masa madre es INVERSAMENTE proporcional a
#    la temperatura ambiente (mas calor -> menos masa madre).
# ============================================================
ZONAS = [
    {"nombre": "Muy frio", "rango": "< 15 C", "limite_inf": None, "limite_sup": 15,
     "ratio": (1, 1, 1), "mm_pct": (30, 35), "maduracion_h": (10, 12),
     "bloque_h": (4.5, 6), "retardo_h": (14, 20), "agua_refresco_c": (28, 32)},
    {"nombre": "Frio", "rango": "15-18 C", "limite_inf": 15, "limite_sup": 18,
     "ratio": (1, 2, 2), "mm_pct": (25, 30), "maduracion_h": (8, 10),
     "bloque_h": (4, 5), "retardo_h": (12, 20), "agua_refresco_c": (26, 30)},
    {"nombre": "Templado", "rango": "18-22 C", "limite_inf": 18, "limite_sup": 22,
     "ratio": (1, 3, 3), "mm_pct": (20, 25), "maduracion_h": (6, 8),
     "bloque_h": (3, 4), "retardo_h": (12, 18), "agua_refresco_c": (22, 26)},
    {"nombre": "Zona neutra", "rango": "22-25 C", "limite_inf": 22, "limite_sup": 25,
     "ratio": (1, 4, 4), "mm_pct": (15, 20), "maduracion_h": (5, 7),
     "bloque_h": (2.5, 3.5), "retardo_h": (12, 15), "agua_refresco_c": None},
    {"nombre": "Calor moderado", "rango": "25-28 C", "limite_inf": 25, "limite_sup": 28,
     "ratio": (1, 4, 4), "mm_pct": (12, 15), "maduracion_h": (4.5, 6),
     "bloque_h": (2, 3), "retardo_h": (12, 18), "agua_refresco_c": None},
    {"nombre": "Calido", "rango": "28-32 C", "limite_inf": 28, "limite_sup": None,
     "ratio": (1, 5, 5), "mm_pct": (8, 10), "maduracion_h": (4.5, 6),
     "bloque_h": (1, 2), "retardo_h": (12, 16), "agua_refresco_c": (10, 15)},
]


def clasificar_zona(temp_ambiente: float) -> dict:
    """Zona termica (ratio, % MM, tiempos) para una temperatura ambiente (C)."""
    for z in ZONAS:
        inf, sup = z["limite_inf"], z["limite_sup"]
        if (inf is None or temp_ambiente >= inf) and (sup is None or temp_ambiente < sup):
            return z
    return ZONAS[-1]


def porcentaje_mm_sugerido(temp_ambiente: float) -> float:
    """% de masa madre sugerido (punto medio del rango de la zona)."""
    lo, hi = clasificar_zona(temp_ambiente)["mm_pct"]
    return (lo + hi) / 2


# ============================================================
# 2) RECETAS (hoja "Calculadora Pan" del Excel)
# ------------------------------------------------------------
# Todos los % son respecto de la HARINA TOTAL (100%).
#   hidratacion = agua total (incluye el agua de la masa madre)
#   integral / centeno = parte de la harina total que es de ese tipo
#   semillas / aceite  = ingredientes EXTRA (suman peso aparte)
# El % de masa madre NO se guarda aca: lo define la temperatura ambiente.
# Valores marcados con (*) fueron completados con un tipico razonable.
# ============================================================
RECETAS = {
    "pan rustico": {
        "nombre": "Pan Rustico", "hidratacion": 78, "sal": 2,
        "integral": 10, "centeno": 5, "peso_sugerido_g": 864,
    },
    "rustico semillas": {
        "nombre": "Rustico Semillas", "hidratacion": 78, "sal": 2,
        "integral": 10, "centeno": 5, "semillas": 13, "peso_sugerido_g": 864,
    },
    "integral": {
        "nombre": "Integral", "hidratacion": 80, "sal": 2,
        "integral": 60, "centeno": 0, "peso_sugerido_g": 874,
    },
    "baguette": {
        "nombre": "Baguette", "hidratacion": 70, "sal": 2,
        "integral": 0, "centeno": 0, "peso_sugerido_g": 250,  # (*) peso tipico
    },
    "focaccia": {
        "nombre": "Focaccia", "hidratacion": 80, "sal": 2,  # (*) sal tipica
        "integral": 10, "centeno": 0, "aceite": 5,           # (*) aceite tipico
    },
}

# Panes que todavia NO estan cargados (necesitan su receta completa):
#   - Pan de Molde: enriquecido (manteca, leche) -> modelo distinto.
#   - Croissant: laminado (empaste de manteca) -> modelo distinto.


def listar_recetas() -> list:
    """Lista los panes disponibles con su resumen."""
    return [
        {"clave": k, "nombre": r["nombre"], "hidratacion_pct": r["hidratacion"],
         "integral_pct": r.get("integral", 0), "centeno_pct": r.get("centeno", 0),
         "sal_pct": r["sal"], "semillas_pct": r.get("semillas", 0),
         "aceite_pct": r.get("aceite", 0)}
        for k, r in RECETAS.items()
    ]


def obtener_receta(nombre: str) -> Optional[dict]:
    """Busca una receta por nombre (tolerante a mayusculas/acentos simples)."""
    if not nombre:
        return None
    clave = nombre.strip().lower()
    if clave in RECETAS:
        return RECETAS[clave]
    # Busqueda flexible por coincidencia parcial.
    for k, r in RECETAS.items():
        if clave in k or k in clave or clave in r["nombre"].lower():
            return r
    return None


# ============================================================
# 3) CALCULO DE INGREDIENTES (modelo igual al Excel)
# ============================================================
def calcular_ingredientes(n_piezas: int, peso_pieza_g: float, hidratacion_pct: float,
                          sal_pct: float, mm_pct: float, integral_pct: float = 0.0,
                          centeno_pct: float = 0.0, semillas_pct: float = 0.0,
                          aceite_pct: float = 0.0, mm_hidratacion_pct: float = 100.0) -> dict:
    """Calcula los ingredientes por pieza y total.

    Modelo (igual al Excel):
      - La masa madre (prefermento) ya tiene su harina y su agua DENTRO de la
        harina total y de la hidratacion: NO se suma aparte.
      - semillas y aceite SI suman peso extra.
      - mm_hidratacion_pct: hidratacion del prefermento (100 = mitad harina, mitad agua).
    """
    extras = semillas_pct + aceite_pct
    masa_total = n_piezas * peso_pieza_g

    # De la masa total se despeja la harina total F.
    factor = 1 + (hidratacion_pct + sal_pct + extras) / 100
    F = masa_total / factor

    # Masa madre (prefermento) interna.
    mm_total = F * mm_pct / 100
    # Reparto harina/agua del prefermento segun su hidratacion.
    mm_agua = mm_total * (mm_hidratacion_pct / 100) / (1 + mm_hidratacion_pct / 100)
    mm_harina = mm_total - mm_agua

    # Aguas y harinas que se agregan en el amasijo (descontando lo que ya esta en la MM).
    agua_total = F * hidratacion_pct / 100
    agua_amasijo = agua_total - mm_agua

    harina_integral = F * integral_pct / 100
    harina_centeno = F * centeno_pct / 100
    harina_blanca = F - harina_integral - harina_centeno - mm_harina

    sal = F * sal_pct / 100
    semillas = F * semillas_pct / 100
    aceite = F * aceite_pct / 100

    ingredientes_total = {"masa_madre_g": round(mm_total, 1)}
    if harina_blanca > 0.05:
        ingredientes_total["harina_blanca_g"] = round(harina_blanca, 1)
    if harina_integral > 0.05:
        ingredientes_total["harina_integral_g"] = round(harina_integral, 1)
    if harina_centeno > 0.05:
        ingredientes_total["harina_centeno_g"] = round(harina_centeno, 1)
    ingredientes_total["agua_g"] = round(agua_amasijo, 1)
    ingredientes_total["sal_g"] = round(sal, 1)
    if semillas > 0.05:
        ingredientes_total["semillas_g"] = round(semillas, 1)
    if aceite > 0.05:
        ingredientes_total["aceite_g"] = round(aceite, 1)

    ingredientes_por_pieza = {k: round(v / n_piezas, 1) for k, v in ingredientes_total.items()}

    return {
        "n_piezas": n_piezas,
        "peso_pieza_g": peso_pieza_g,
        "masa_total_g": round(masa_total, 1),
        "harina_total_g": round(F, 1),
        "agua_total_g": round(agua_total, 1),
        "masa_madre_necesaria_g": round(mm_total, 1),
        "porcentajes": {
            "hidratacion_pct": hidratacion_pct, "sal_pct": sal_pct, "mm_pct": mm_pct,
            "integral_pct": integral_pct, "centeno_pct": centeno_pct,
            "semillas_pct": semillas_pct, "aceite_pct": aceite_pct,
        },
        "ingredientes_total": ingredientes_total,
        "ingredientes_por_pieza": ingredientes_por_pieza,
    }


def calcular_produccion(pan: str, n_piezas: int, peso_pieza_g: Optional[float],
                        temp_ambiente: float, mm_pct: Optional[float] = None) -> dict:
    """Calculo COMPLETO de una produccion: junta receta + temperatura.

    - Toma la receta del pan (hidratacion, integral, sal, etc.).
    - Clasifica la zona por temperatura y, si no se indica mm_pct, usa el % de
      masa madre sugerido por la temperatura (punto medio del rango).
    - Devuelve ingredientes (por pieza y total), la zona y la masa madre necesaria.
    """
    receta = obtener_receta(pan)
    if receta is None:
        return {"error": f"No tengo cargada la receta de '{pan}'.",
                "recetas_disponibles": [r["nombre"] for r in listar_recetas()]}

    zona = clasificar_zona(temp_ambiente)
    if mm_pct is None:
        mm_pct = porcentaje_mm_sugerido(temp_ambiente)
    if peso_pieza_g is None:
        peso_pieza_g = receta.get("peso_sugerido_g")
        if peso_pieza_g is None:
            return {"error": f"Falta el peso por pieza para '{receta['nombre']}'."}

    ingr = calcular_ingredientes(
        n_piezas, peso_pieza_g, receta["hidratacion"], receta["sal"], mm_pct,
        integral_pct=receta.get("integral", 0), centeno_pct=receta.get("centeno", 0),
        semillas_pct=receta.get("semillas", 0), aceite_pct=receta.get("aceite", 0))

    return {
        "pan": receta["nombre"],
        "temp_ambiente": temp_ambiente,
        "zona": {"nombre": zona["nombre"], "rango": zona["rango"],
                 "ratio_refresco": f"1:{zona['ratio'][1]}:{zona['ratio'][2]}",
                 "mm_pct_usado": round(mm_pct, 1), "mm_pct_rango": zona["mm_pct"],
                 "bloque_h": zona["bloque_h"], "maduracion_h": zona["maduracion_h"],
                 "retardo_h": zona["retardo_h"]},
        **ingr,
    }


# ============================================================
# 4) FORMULA DE LA SEMILLA (cuanta masa madre partir)
#    M0 = Mf / (1 + P_H + P_Ag)^k
# ============================================================
def calcular_masa_madre_semilla(masa_final_g: float, p_harina: float,
                                p_agua: float, refrescos: int) -> dict:
    """Cuanta masa madre semilla (M0) partir para llegar, luego de 'refrescos'
    refrescos, a la masa madre final deseada (masa_final_g)."""
    if refrescos < 1:
        raise ValueError("Debe haber al menos 1 refresco.")
    divisor = (1 + p_harina + p_agua) ** refrescos
    m0 = masa_final_g / divisor

    pasos = []
    masa = m0
    for i in range(1, refrescos + 1):
        harina = p_harina * masa
        agua = p_agua * masa
        masa_res = masa + harina + agua
        pasos.append({"refresco": i, "masa_inicial_g": round(masa, 1),
                      "harina_g": round(harina, 1), "agua_g": round(agua, 1),
                      "masa_resultante_g": round(masa_res, 1)})
        masa = masa_res

    return {"masa_madre_semilla_g": round(m0, 1), "masa_madre_final_g": round(masa, 1),
            "ratio": f"1:{p_harina:g}:{p_agua:g}", "refrescos": refrescos, "pasos": pasos}


# ============================================================
# 5) TEMPERATURA DEL AGUA
#    T_agua = 4*T_obj - (T_harina + T_amb + T_mm + F)
# ============================================================
def temperatura_agua(temp_objetivo: float, temp_harina: float, temp_ambiente: float,
                     temp_masamadre: float, metodo: str = "amasadora") -> dict:
    """Temperatura del agua para llegar a la temperatura objetivo de la masa.
    'metodo': 'amasadora' (F=4) o 'mano' (F=2)."""
    metodo = (metodo or "").strip().lower()
    if metodo in ("mano", "a mano", "manual"):
        friccion, metodo_nombre = 2, "a mano"
    else:
        friccion, metodo_nombre = 4, "amasadora"
    t_agua = 4 * temp_objetivo - (temp_harina + temp_ambiente + temp_masamadre + friccion)
    return {"temp_agua_c": round(t_agua, 1), "friccion_usada_c": friccion,
            "metodo": metodo_nombre,
            "detalle": (f"4*{temp_objetivo} - ({temp_harina} + {temp_ambiente} + "
                        f"{temp_masamadre} + {friccion}) = {round(t_agua, 1)} C")}


# ============================================================
# DEMO / PRUEBA RAPIDA
# ============================================================
if __name__ == "__main__":
    print("== Produccion: 1 Pan Rustico de 864 g a 24 C ==")
    prod = calcular_produccion("pan rustico", 1, 864, 24, mm_pct=20)
    print("Harina total:", prod["harina_total_g"], "g | Agua total:", prod["agua_total_g"], "g")
    print("Ingredientes (amasijo):", prod["ingredientes_total"])
    print("-> Esperado del Excel: blanca 360, integral 48, centeno 24, agua 326.4, sal 9.6, MM 96")

    print("\n== Produccion: 3 Integral de ~874 g a 24 C (mm 20%) ==")
    prod2 = calcular_produccion("integral", 3, 873.6, 24, mm_pct=20)
    print("Harina total:", prod2["harina_total_g"], "Ingredientes:", prod2["ingredientes_total"])

    print("\n== Masa madre semilla (ratio 1:4:4, 2 refrescos) para 96 g ==")
    print(calcular_masa_madre_semilla(96, 4, 4, 2)["pasos"])

    print("\n== Temperatura del agua (obj 24, harina 22, amb 24, MM 23, amasadora) ==")
    print(temperatura_agua(24, 22, 24, 23, "amasadora")["temp_agua_c"], "C")
