-- ============================================================
-- Análisis de cartera de créditos - FinCore S.A.
-- Autora: Kellys Tarache
-- Motor: MySQL 8.0+
--
-- Construye la capa de análisis y control de una cartera de
-- créditos: clasificación de mora, vistas, auditoría automática,
-- procedimientos transaccionales, reportes y optimización.
--
-- Orden de ejecución:
--   1. sql/01_datos_fincore.sql   (crea la base y carga datos)
--   2. sql/02_solucion_fincore.sql (este archivo)
-- ============================================================

USE fincore_sa;

-- ============================================================
-- PARTE 1 - FUNCIÓN DE CLASIFICACIÓN DE MORA
-- ============================================================

DROP FUNCTION IF EXISTS fn_clasificar_mora;

DELIMITER //

CREATE FUNCTION fn_clasificar_mora(dias_mora INT)
RETURNS VARCHAR(30)
DETERMINISTIC
BEGIN
    IF dias_mora <= 0 THEN
        RETURN 'Al día';
    ELSEIF dias_mora BETWEEN 1 AND 30 THEN
        RETURN 'Mora Temprana (1-30d)';
    ELSEIF dias_mora BETWEEN 31 AND 60 THEN
        RETURN 'Mora Media (31-60d)';
    ELSEIF dias_mora BETWEEN 61 AND 90 THEN
        RETURN 'Mora Grave (61-90d)';
    ELSE
        RETURN 'Mora Crítica (+90d)';
    END IF;
END //

DELIMITER ;

-- ============================================================
-- PARTE 2 - VISTAS DE ANÁLISIS
-- ============================================================

DROP VIEW IF EXISTS vw_estado_cartera;
DROP VIEW IF EXISTS vw_resumen_clientes;

-- Vista: estado detallado de cada préstamo
CREATE VIEW vw_estado_cartera AS
SELECT
    pt.id_prestamo,
    c.nombre AS cliente,
    c.segmento,
    c.categoria_riesgo,
    pd.nombre AS producto,
    pt.monto_otorgado,
    pt.saldo_pendiente,
    pt.tasa_aplicada,
    pt.estado,
    ROUND(
        (pt.saldo_pendiente / NULLIF(pt.monto_otorgado, 0)) * 100,
        2
    ) AS porcentaje_saldo_restante,
    pt.dias_mora,
    fn_clasificar_mora(pt.dias_mora) AS categoria_mora
FROM clientes c
JOIN prestamos pt
    ON c.id_cliente = pt.id_cliente
JOIN productos_credito pd
    ON pd.id_producto = pt.id_producto;

-- Vista: resumen consolidado por cliente
CREATE VIEW vw_resumen_clientes AS
SELECT
    c.id_cliente,
    c.nombre AS cliente,
    COUNT(pt.id_prestamo) AS total_prestamos,
    COALESCE(SUM(pt.monto_otorgado), 0) AS total_monto_otorgado,
    COALESCE(SUM(pt.saldo_pendiente), 0) AS total_saldo_pendiente_consolidado,
    COALESCE(MAX(pt.dias_mora), 0) AS max_dias_mora,
    CASE
        WHEN MAX(pt.dias_mora) > 0 THEN 'Sí'
        ELSE 'No'
    END AS mora_activa
FROM clientes c
LEFT JOIN prestamos pt
    ON c.id_cliente = pt.id_cliente
GROUP BY
    c.id_cliente,
    c.nombre;

-- ============================================================
-- PARTE 3 - TRIGGER DE AUDITORÍA
-- ============================================================

DROP TRIGGER IF EXISTS trg_auditoria_estado;

DELIMITER //

CREATE TRIGGER trg_auditoria_estado
AFTER UPDATE ON prestamos
FOR EACH ROW
BEGIN

    -- Registrar cambios en el estado del préstamo
    IF NOT (OLD.estado <=> NEW.estado) THEN
        INSERT INTO auditoria_prestamos (
            id_prestamo,
            campo_modificado,
            valor_anterior,
            valor_nuevo,
            fecha_cambio
        )
        VALUES (
            NEW.id_prestamo,
            'estado',
            OLD.estado,
            NEW.estado,
            NOW()
        );
    END IF;

    -- Registrar cambios en los días de mora
    IF NOT (OLD.dias_mora <=> NEW.dias_mora) THEN
        INSERT INTO auditoria_prestamos (
            id_prestamo,
            campo_modificado,
            valor_anterior,
            valor_nuevo,
            fecha_cambio
        )
        VALUES (
            NEW.id_prestamo,
            'dias_mora',
            OLD.dias_mora,
            NEW.dias_mora,
            NOW()
        );
    END IF;

END //

DELIMITER ;

-- ============================================================
-- PARTE 4 - PROCEDIMIENTOS ALMACENADOS
-- ============================================================

-- ------------------------------------------------------------
-- 4.1 Registrar pago
-- ------------------------------------------------------------

DROP PROCEDURE IF EXISTS sp_registrar_pago;

DELIMITER //

CREATE PROCEDURE sp_registrar_pago(
    IN p_id_prestamo INT,
    IN p_monto DECIMAL(10,2),
    IN p_tipo_pago VARCHAR(20),
    IN p_canal VARCHAR(30)
)
BEGIN
    DECLARE v_saldo DECIMAL(12,2);
    DECLARE v_estado VARCHAR(20);
    DECLARE v_existe INT DEFAULT 0;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;

    -- Verificar existencia del préstamo
    SELECT COUNT(*)
    INTO v_existe
    FROM prestamos
    WHERE id_prestamo = p_id_prestamo;

    IF v_existe = 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'El préstamo no existe.';
    END IF;

    -- Bloquear el préstamo y obtener sus datos actuales
    SELECT saldo_pendiente, estado
    INTO v_saldo, v_estado
    FROM prestamos
    WHERE id_prestamo = p_id_prestamo
    FOR UPDATE;

    -- Solo se permiten pagos para préstamos activos o vencidos
    IF v_estado NOT IN ('Activo', 'Vencido') THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'El préstamo no está Activo ni Vencido.';
    END IF;

    -- Validar monto positivo
    IF p_monto <= 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'El monto del pago debe ser mayor que cero.';
    END IF;

    -- Evitar pagos superiores al saldo
    IF p_monto > v_saldo THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'El monto del pago supera el saldo pendiente.';
    END IF;

    -- Descontar el pago del saldo pendiente
    UPDATE prestamos
    SET saldo_pendiente = saldo_pendiente - p_monto
    WHERE id_prestamo = p_id_prestamo;

    -- Registrar el pago
    INSERT INTO pagos (
        id_prestamo,
        fecha_pago,
        monto_pagado,
        tipo_pago,
        canal
    )
    VALUES (
        p_id_prestamo,
        CURDATE(),
        p_monto,
        p_tipo_pago,
        p_canal
    );

    -- Si la deuda queda en cero, cancelar el préstamo
    IF v_saldo - p_monto = 0 THEN
        UPDATE prestamos
        SET estado = 'Cancelado'
        WHERE id_prestamo = p_id_prestamo;
    END IF;

    COMMIT;
END //

DELIMITER ;

-- ------------------------------------------------------------
-- 4.2 Generar resumen mensual
-- ------------------------------------------------------------

DROP PROCEDURE IF EXISTS sp_generar_resumen_mensual;

DELIMITER //

CREATE PROCEDURE sp_generar_resumen_mensual(
    IN p_anio INT,
    IN p_mes INT
)
BEGIN
    DECLARE v_total_prestamos INT DEFAULT 0;
    DECLARE v_monto_total DECIMAL(14,2) DEFAULT 0;
    DECLARE v_total_pagos DECIMAL(14,2) DEFAULT 0;
    DECLARE v_prestamos_vencidos INT DEFAULT 0;
    DECLARE v_tasa_morosidad DECIMAL(5,2) DEFAULT 0;

    -- Préstamos otorgados en el período
    SELECT
        COUNT(*),
        COALESCE(SUM(monto_otorgado), 0)
    INTO
        v_total_prestamos,
        v_monto_total
    FROM prestamos
    WHERE YEAR(fecha_otorgamiento) = p_anio
      AND MONTH(fecha_otorgamiento) = p_mes;

    -- Pagos recibidos en el período
    SELECT
        COALESCE(SUM(monto_pagado), 0)
    INTO v_total_pagos
    FROM pagos
    WHERE YEAR(fecha_pago) = p_anio
      AND MONTH(fecha_pago) = p_mes;

    -- Préstamos con mora dentro del período
    SELECT
        COUNT(*)
    INTO v_prestamos_vencidos
    FROM prestamos
    WHERE YEAR(fecha_otorgamiento) = p_anio
      AND MONTH(fecha_otorgamiento) = p_mes
      AND dias_mora > 0;

    -- Calcular tasa de morosidad
    IF v_total_prestamos > 0 THEN
        SET v_tasa_morosidad =
            ROUND((v_prestamos_vencidos / v_total_prestamos) * 100, 2);
    END IF;

    -- Insertar o actualizar el resumen del período
    INSERT INTO resumen_mensual (
        anio,
        mes,
        total_prestamos,
        monto_total_otorgado,
        total_pagos_recibidos,
        prestamos_vencidos,
        tasa_morosidad
    )
    VALUES (
        p_anio,
        p_mes,
        v_total_prestamos,
        v_monto_total,
        v_total_pagos,
        v_prestamos_vencidos,
        v_tasa_morosidad
    )
    ON DUPLICATE KEY UPDATE
        total_prestamos = VALUES(total_prestamos),
        monto_total_otorgado = VALUES(monto_total_otorgado),
        total_pagos_recibidos = VALUES(total_pagos_recibidos),
        prestamos_vencidos = VALUES(prestamos_vencidos),
        tasa_morosidad = VALUES(tasa_morosidad),
        fecha_generacion = CURRENT_TIMESTAMP;
END //

DELIMITER ;

-- ------------------------------------------------------------
-- 4.3 Refinanciar préstamo
-- ------------------------------------------------------------

DROP PROCEDURE IF EXISTS sp_refinanciar_prestamo;

DELIMITER //

CREATE PROCEDURE sp_refinanciar_prestamo(
    IN p_id_prestamo INT,
    IN p_nuevo_plazo INT,
    IN p_nueva_tasa DECIMAL(5,2)
)
BEGIN
    DECLARE v_id_cliente INT;
    DECLARE v_id_producto INT;
    DECLARE v_saldo DECIMAL(12,2);
    DECLARE v_estado VARCHAR(20);
    DECLARE v_cuota DECIMAL(10,2);
    DECLARE v_existe INT DEFAULT 0;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;

    -- Verificar existencia del préstamo
    SELECT COUNT(*)
    INTO v_existe
    FROM prestamos
    WHERE id_prestamo = p_id_prestamo;

    IF v_existe = 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'El préstamo no existe.';
    END IF;

    -- Obtener información y bloquear el préstamo
    SELECT
        id_cliente,
        id_producto,
        saldo_pendiente,
        estado
    INTO
        v_id_cliente,
        v_id_producto,
        v_saldo,
        v_estado
    FROM prestamos
    WHERE id_prestamo = p_id_prestamo
    FOR UPDATE;

    -- Solo se pueden refinanciar préstamos activos o vencidos
    IF v_estado NOT IN ('Activo', 'Vencido') THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'El préstamo no puede ser refinanciado.';
    END IF;

    -- Validar parámetros
    IF p_nuevo_plazo <= 0 OR p_nueva_tasa < 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'El plazo o la tasa no son válidos.';
    END IF;

    -- Cuota mensual aproximada según saldo, tasa y nuevo plazo
    SET v_cuota =
        v_saldo *
        (1 + (p_nueva_tasa / 100) / 12 * p_nuevo_plazo)
        / p_nuevo_plazo;

    -- Marcar el préstamo original como refinanciado
    UPDATE prestamos
    SET estado = 'Refinanciado'
    WHERE id_prestamo = p_id_prestamo;

    -- Crear el nuevo préstamo con el saldo pendiente como monto
    INSERT INTO prestamos (
        id_cliente,
        id_producto,
        plazo_meses,
        dias_mora,
        estado,
        tasa_aplicada,
        monto_otorgado,
        saldo_pendiente,
        cuota_mensual,
        fecha_otorgamiento,
        fecha_vencimiento
    )
    VALUES (
        v_id_cliente,
        v_id_producto,
        p_nuevo_plazo,
        0,
        'Activo',
        p_nueva_tasa,
        v_saldo,
        v_saldo,
        ROUND(v_cuota, 2),
        CURDATE(),
        DATE_ADD(CURDATE(), INTERVAL p_nuevo_plazo MONTH)
    );

    COMMIT;
END //

DELIMITER ;

-- ============================================================
-- PARTE 5 - CONSULTAS ANALÍTICAS
-- ============================================================

-- 5.1 Aging de cartera
-- Préstamos Activos/Vencidos agrupados por tramo de mora.

SELECT
    fn_clasificar_mora(dias_mora) AS tramo_mora,
    COUNT(*) AS cantidad_prestamos,
    SUM(saldo_pendiente) AS saldo_en_riesgo,
    ROUND(
        SUM(saldo_pendiente) /
        NULLIF(
            (
                SELECT SUM(saldo_pendiente)
                FROM prestamos
                WHERE estado IN ('Activo', 'Vencido')
            ),
            0
        ) * 100,
        2
    ) AS porcentaje_total
FROM prestamos
WHERE estado IN ('Activo', 'Vencido')
GROUP BY fn_clasificar_mora(dias_mora)
ORDER BY MIN(dias_mora);

-- 5.2 Top 5 de exposición crediticia
-- Compara los 5 clientes con mayor saldo contra el promedio
-- de exposición de todos los clientes con préstamos.

WITH exposicion AS (
    SELECT
        c.id_cliente,
        c.nombre AS cliente,
        SUM(p.saldo_pendiente) AS saldo_total
    FROM clientes c
    JOIN prestamos p
        ON c.id_cliente = p.id_cliente
    GROUP BY
        c.id_cliente,
        c.nombre
),
promedio AS (
    SELECT AVG(saldo_total) AS promedio_cartera
    FROM exposicion
)
SELECT
    e.id_cliente,
    e.cliente,
    e.saldo_total,
    ROUND(p.promedio_cartera, 2) AS promedio_cartera,
    ROUND(
        e.saldo_total - p.promedio_cartera,
        2
    ) AS diferencia_promedio
FROM exposicion e
CROSS JOIN promedio p
ORDER BY e.saldo_total DESC
LIMIT 5;

-- 5.3 Historial completo de pagos
-- Últimos 20 pagos registrados con préstamo, cliente y producto.

SELECT
    pg.id_pago,
    pg.fecha_pago,
    pg.monto_pagado,
    pg.tipo_pago,
    pg.canal,
    p.id_prestamo,
    c.nombre AS cliente,
    pc.nombre AS producto
FROM pagos pg
JOIN prestamos p
    ON pg.id_prestamo = p.id_prestamo
JOIN clientes c
    ON p.id_cliente = c.id_cliente
JOIN productos_credito pc
    ON p.id_producto = pc.id_producto
ORDER BY
    pg.fecha_pago DESC,
    pg.id_pago DESC
LIMIT 20;

-- 5.4 Reporte ejecutivo
-- Agrupa por segmento y categoría de riesgo.

SELECT
    segmento,
    categoria_riesgo,
    ROUND(AVG(tasa_aplicada), 2) AS tasa_promedio,
    SUM(
        CASE
            WHEN estado = 'Vencido' THEN 1
            ELSE 0
        END
    ) AS creditos_vencidos,
    ROUND(
        SUM(
            CASE
                WHEN dias_mora > 0 THEN saldo_pendiente
                ELSE 0
            END
        ) /
        NULLIF(SUM(saldo_pendiente), 0) * 100,
        2
    ) AS porcentaje_cartera_en_riesgo
FROM vw_estado_cartera
GROUP BY
    segmento,
    categoria_riesgo
ORDER BY
    segmento,
    categoria_riesgo;

-- ============================================================
-- PARTE 6 - OPTIMIZACIÓN
-- ============================================================

-- EXPLAIN de referencia para la consulta de historial de pagos.
-- En la ejecución original, pagos aparece como tabla conductora.
-- La consulta utiliza las PK de las tablas relacionadas.

EXPLAIN FORMAT=TRADITIONAL
SELECT
    pg.id_pago,
    pg.fecha_pago,
    pg.monto_pagado,
    pg.tipo_pago,
    pg.canal,
    p.id_prestamo,
    c.nombre AS cliente,
    pc.nombre AS producto
FROM pagos pg
JOIN prestamos p
    ON pg.id_prestamo = p.id_prestamo
JOIN clientes c
    ON p.id_cliente = c.id_cliente
JOIN productos_credito pc
    ON p.id_producto = pc.id_producto
ORDER BY
    pg.fecha_pago DESC,
    pg.id_pago DESC
LIMIT 20;

-- Índices adicionales sobre columnas utilizadas en filtros, JOIN
-- y ordenamiento.

-- MySQL no soporta CREATE INDEX IF NOT EXISTS, así que se usa un
-- procedimiento auxiliar que crea el índice solo si aún no existe.
-- Así el script se puede ejecutar más de una vez sin error 1061.

DROP PROCEDURE IF EXISTS sp_crear_indice;

DELIMITER //

CREATE PROCEDURE sp_crear_indice(
    IN p_tabla VARCHAR(64),
    IN p_indice VARCHAR(64),
    IN p_columnas VARCHAR(255)
)
BEGIN
    IF NOT EXISTS (
        SELECT 1
        FROM information_schema.statistics
        WHERE table_schema = DATABASE()
          AND table_name = p_tabla
          AND index_name = p_indice
    ) THEN
        SET @sql_idx = CONCAT('CREATE INDEX ', p_indice,
                              ' ON ', p_tabla, ' (', p_columnas, ')');
        PREPARE stmt_idx FROM @sql_idx;
        EXECUTE stmt_idx;
        DEALLOCATE PREPARE stmt_idx;
    END IF;
END //

DELIMITER ;

CALL sp_crear_indice('pagos', 'idx_pagos_fecha_id', 'fecha_pago DESC, id_pago DESC');
CALL sp_crear_indice('pagos', 'idx_pagos_prestamo_fecha', 'id_prestamo, fecha_pago');
CALL sp_crear_indice('prestamos', 'idx_prestamos_cliente_producto', 'id_cliente, id_producto');

DROP PROCEDURE IF EXISTS sp_crear_indice;

-- EXPLAIN posterior a la creación de índices.

EXPLAIN FORMAT=TRADITIONAL
SELECT
    pg.id_pago,
    pg.fecha_pago,
    pg.monto_pagado,
    pg.tipo_pago,
    pg.canal,
    p.id_prestamo,
    c.nombre AS cliente,
    pc.nombre AS producto
FROM pagos pg
JOIN prestamos p
    ON pg.id_prestamo = p.id_prestamo
JOIN clientes c
    ON p.id_cliente = c.id_cliente
JOIN productos_credito pc
    ON p.id_producto = pc.id_producto
ORDER BY
    pg.fecha_pago DESC,
    pg.id_pago DESC
LIMIT 20;

-- ============================================================
-- PARTE 7 - EVIDENCIA DE EJECUCIÓN (TRIGGER Y RESUMEN MENSUAL)
-- ============================================================
-- Estas llamadas se ejecutan junto con el script para dejar
-- evidencia en auditoria_prestamos y resumen_mensual.

-- ------------------------------------------------------------
-- 7.1 Prueba del trigger trg_auditoria_estado
-- ------------------------------------------------------------
-- Se guardan los valores originales del préstamo, se modifican
-- estado y dias_mora (el trigger registra 1 fila por campo) y
-- luego se restauran los valores originales (2 filas más).
-- Resultado esperado: al menos 4 registros en auditoria_prestamos,
-- y la cartera queda con sus datos originales.

SELECT estado, dias_mora
INTO @estado_original, @mora_original
FROM prestamos
WHERE id_prestamo = 2;

UPDATE prestamos
SET estado = 'Vencido',
    dias_mora = 45
WHERE id_prestamo = 2;

UPDATE prestamos
SET estado = @estado_original,
    dias_mora = @mora_original
WHERE id_prestamo = 2;

SELECT * FROM auditoria_prestamos ORDER BY id_auditoria;

-- ------------------------------------------------------------
-- 7.2 Ejecución de sp_generar_resumen_mensual
-- ------------------------------------------------------------
-- Se generan 3 períodos distintos (mínimo requerido).
-- Ajustar año/mes si los datos de prestamos están en otras fechas.

CALL sp_generar_resumen_mensual(2023, 1);
CALL sp_generar_resumen_mensual(2023, 2);
CALL sp_generar_resumen_mensual(2023, 3);

SELECT * FROM resumen_mensual ORDER BY anio, mes;

-- ------------------------------------------------------------
-- 7.3 Verificación de mínimos
-- ------------------------------------------------------------

SELECT 'auditoria_prestamos' AS tabla, COUNT(*) AS registros FROM auditoria_prestamos
UNION ALL
SELECT 'resumen_mensual', COUNT(*) FROM resumen_mensual;

-- ============================================================
-- PRUEBAS MANUALES OPCIONALES (modifican datos)
-- ============================================================
-- Para probar el pago:
-- CALL sp_registrar_pago(2, 500.00, 'Cuota', 'App');
--
-- Para probar refinanciación:
-- CALL sp_refinanciar_prestamo(14, 24, 20.00);
--
-- ============================================================
-- FIN DEL SCRIPT
-- ============================================================
