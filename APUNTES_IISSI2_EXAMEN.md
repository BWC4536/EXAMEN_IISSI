# 📚 APUNTES DEFINITIVOS — IISSI2 Backend
**Asignatura:** Ingeniería e Integración de Sistemas de Información 2  
**Fecha:** Examen mañana  
**Repos:** Lab1 · Lab2 · Lab3 · Examen Schedules · Examen ShippingAddress · Examen Reviews

---

## 🚨 RESUMEN EJECUTIVO — LO QUE SÍ CAE

Los 3 exámenes siguen **exactamente el mismo patrón**: añadir una nueva entidad (Schedule, ShippingAddress, Review) al proyecto DeliverUS implementando siempre los mismos 5 bloques en el mismo orden:

```
1. MIGRACIÓN    → define la tabla en la BD
2. MODELO       → define la clase Sequelize con campos y relaciones
3. RUTAS        → define endpoints + middlewares
4. VALIDACIÓN   → reglas de express-validator
5. CONTROLADOR  → lógica de negocio (CRUD)
   + a veces: MIDDLEWARE propio (ownership, checks de negocio)
   + a veces: CONSULTA ESPECIAL (filtrar por condición)
```

---

## 📋 ÍNDICE

1. [Migración](#1-migración)
2. [Modelo](#2-modelo)
3. [Rutas y Middlewares](#3-rutas-y-middlewares)
4. [Validación](#4-validación)
5. [Controlador](#5-controlador)
6. [Middleware propio](#6-middleware-propio)
7. [Consultas especiales](#7-consultas-especiales)
8. [Atributos virtuales](#8-atributos-virtuales)
9. [Errores comunes](#9-errores-comunes)
10. [Código rápido de referencia](#10-código-rápido-de-referencia)

---

## 1. MIGRACIÓN

### Concepto
Define la estructura de la tabla en MariaDB. Cada campo necesita tipo, constraints y FK si referencia otra tabla.

### Plantilla completa
```javascript
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('NombreTablas', {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER
      },
      // Campos de texto
      nombre: {
        allowNull: false,      // NOT NULL
        type: Sequelize.STRING
      },
      descripcion: {
        type: Sequelize.TEXT   // opcional → sin allowNull
      },
      // Campos numéricos
      precio: {
        allowNull: false,
        type: Sequelize.DOUBLE
      },
      // Campos booleanos
      esDefault: {
        allowNull: false,
        defaultValue: false,
        type: Sequelize.BOOLEAN
      },
      // Campos de hora (HH:mm:ss)
      startTime: {
        allowNull: false,
        type: Sequelize.TIME
      },
      // Clave foránea
      restaurantId: {
        allowNull: false,
        type: Sequelize.INTEGER,
        references: { model: { tableName: 'Restaurants' }, key: 'id' },
        onDelete: 'CASCADE'    // si se borra el padre, se borran los hijos
      },
      // FK opcional (allowNull: true + SET NULL)
      scheduleId: {
        allowNull: true,
        type: Sequelize.INTEGER,
        references: { model: { tableName: 'Schedules' }, key: 'id' },
        onDelete: 'SET NULL'   // si se borra el padre, el hijo queda null
      },
      createdAt: { allowNull: false, type: Sequelize.DATE, defaultValue: new Date() },
      updatedAt: { allowNull: false, type: Sequelize.DATE, defaultValue: new Date() }
    })
  },
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('NombreTablas')
  }
}
```

### Tipos de datos más usados
| Sequelize | Uso |
|-----------|-----|
| `STRING` | texto corto (VARCHAR) |
| `TEXT` | texto largo |
| `INTEGER` | entero |
| `DOUBLE` | decimal |
| `BOOLEAN` | true/false |
| `TIME` | HH:mm:ss |
| `DATE` | fecha+hora |
| `ENUM` | valores fijos |

### ⚠️ Errores comunes en migración
- Olvidar `allowNull: false, autoIncrement: true, primaryKey: true` en el `id`
- Nombre de tabla referenciada en plural y con mayúscula: `'Restaurants'` no `'restaurant'`
- Olvidar `createdAt` y `updatedAt`
- `CASCADE` vs `SET NULL`: CASCADE borra el hijo, SET NULL deja el campo a null

### Modificar migración existente (añadir campo a tabla ya creada)
```javascript
// En lugar de createTable, usar addColumn
await queryInterface.addColumn('Products', 'scheduleId', {
  allowNull: true,
  type: Sequelize.INTEGER,
  references: { model: { tableName: 'Schedules' }, key: 'id' },
  onDelete: 'SET NULL'
})
```

---

## 2. MODELO

### Concepto
Define la clase Sequelize con los campos, relaciones y métodos. Es lo que usas en el controlador.

### Plantilla completa
```javascript
import { Model } from 'sequelize'

const loadModel = (sequelize, DataTypes) => {
  class NombreEntidad extends Model {
    static associate (models) {
      // Relaciones — ver tabla abajo
      NombreEntidad.belongsTo(models.Restaurant, { foreignKey: 'restaurantId', as: 'restaurant' })
      NombreEntidad.hasMany(models.Product, { foreignKey: 'entidadId', as: 'products' })
    }
  }

  NombreEntidad.init({
    campo: {
      allowNull: false,
      type: DataTypes.STRING
    },
    campoOpcional: DataTypes.STRING,    // forma corta para opcionales
    esDefault: {
      allowNull: false,
      defaultValue: false,
      type: DataTypes.BOOLEAN
    },
    foreignKeyId: {
      allowNull: false,
      type: DataTypes.INTEGER
    }
  }, {
    sequelize,
    modelName: 'NombreEntidad'
  })

  return NombreEntidad
}
export default loadModel
```

### Tabla de relaciones
| Situación | En el modelo que tiene la FK | En el modelo padre |
|-----------|-----------------------------|--------------------|
| Review pertenece a Restaurant | `Review.belongsTo(models.Restaurant, {foreignKey:'restaurantId', as:'restaurant'})` | `Restaurant.hasMany(models.Review, {foreignKey:'restaurantId', as:'reviews'})` |
| Product pertenece a Schedule (opcional) | `Product.belongsTo(models.Schedule, {foreignKey:'scheduleId', as:'schedule'})` | `Schedule.hasMany(models.Product, {foreignKey:'scheduleId', as:'products'})` |
| ShippingAddress pertenece a User | `ShippingAddress.belongsTo(models.User, {foreignKey:'userId', as:'user'})` | `User.hasMany(models.ShippingAddress, {foreignKey:'userId', as:'shippingAddresses'})` |

### Registrar el modelo en models.js
Verifica que el archivo `src/models/models.js` importa y registra tu nuevo modelo. Si no aparece, añádelo siguiendo el patrón de los demás.

### Atributo VIRTUAL (calculado, no en BD)
```javascript
avgStars: {
  type: DataTypes.VIRTUAL,
  get () { return this.getDataValue('avgStars') },
  set (value) { this.setDataValue('avgStars', value) }
}
```

---

## 3. RUTAS Y MIDDLEWARES

### Concepto
Cada ruta = endpoint HTTP. Los middlewares se ejecutan en cadena antes del controlador. El orden importa.

### Cadena de middlewares por tipo de operación
```
GET público:
  checkEntityExists → Controller.index/show

GET privado (solo el dueño):
  isLoggedIn → hasRole('owner') → checkEntityExists → checkOwnership → Controller

POST (crear, solo autenticado):
  isLoggedIn → hasRole('customer'/'owner') → checkEntityExists → Validation → handleValidation → Controller

PUT/PATCH (editar):
  isLoggedIn → hasRole → checkEntityExists(padre) → checkOwnership → checkEntityExists(hijo) → Validation → handleValidation → Controller

DELETE:
  isLoggedIn → hasRole → checkEntityExists(padre) → checkOwnership → checkEntityExists(hijo) → Controller
```

### Plantilla de rutas completa
```javascript
import EntidadController from '../controllers/EntidadController.js'
import * as EntidadValidation from '../controllers/validation/EntidadValidation.js'
import { isLoggedIn, hasRole } from '../middlewares/AuthMiddleware.js'
import { handleValidation } from '../middlewares/ValidationHandlingMiddleware.js'
import { checkEntityExists } from '../middlewares/EntityMiddleware.js'
import * as RestaurantMiddleware from '../middlewares/RestaurantMiddleware.js'
import { Restaurant, Entidad } from '../models/models.js'

const loadEntidadRoutes = function (app) {
  // Colección (sin id de entidad)
  app.route('/restaurants/:restaurantId/entidades')
    .get(
      checkEntityExists(Restaurant, 'restaurantId'),
      EntidadController.index)
    .post(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Restaurant, 'restaurantId'),
      RestaurantMiddleware.checkRestaurantOwnership,
      EntidadValidation.create,
      handleValidation,
      EntidadController.create)

  // Elemento concreto (con id de entidad)
  app.route('/restaurants/:restaurantId/entidades/:entidadId')
    .put(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Restaurant, 'restaurantId'),
      RestaurantMiddleware.checkRestaurantOwnership,
      checkEntityExists(Entidad, 'entidadId'),
      EntidadValidation.update,
      handleValidation,
      EntidadController.update)
    .delete(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Restaurant, 'restaurantId'),
      RestaurantMiddleware.checkRestaurantOwnership,
      checkEntityExists(Entidad, 'entidadId'),
      EntidadController.destroy)
}
export default loadEntidadRoutes
```

### Tabla de middlewares disponibles
| Middleware | Genera | Cuándo usarlo |
|------------|--------|---------------|
| `isLoggedIn` | 401 | Ruta protegida |
| `hasRole('owner'/'customer')` | 403 | Ruta con rol específico |
| `checkEntityExists(Modelo, 'paramId')` | 404 | Verificar que existe el recurso |
| `checkRestaurantOwnership` | 403 | Ruta solo del propietario del restaurante |
| `handleValidation` | 422 | Siempre después de validation |
| `restaurantHasNoOrders` | 409 | Borrar restaurante con pedidos |
| `checkReviewOwnership` | 403 | Ruta solo del autor de la review |
| `checkShippingAddressOwnership` | 403 | Ruta solo del dueño de la dirección |

---

## 4. VALIDACIÓN

### Concepto
Reglas que comprueban los datos del cliente antes de que lleguen al controlador.

### Plantilla y métodos más usados
```javascript
import { check } from 'express-validator'

const create = [
  // Obligatorio string con longitud
  check('name').exists({ checkFalsy: true }).isString().isLength({ min: 1, max: 255 }).trim(),

  // Opcional string
  check('description').optional({ nullable: true, checkFalsy: true }).isString().trim(),

  // Obligatorio número decimal >= 0
  check('price').exists().isFloat({ min: 0 }).toFloat(),

  // Obligatorio entero >= 1 (FK)
  check('restaurantId').exists().isInt({ min: 1 }).toInt(),

  // Obligatorio entero entre 0 y 5 (stars)
  check('stars').exists().isInt({ min: 0, max: 5 }).toInt(),

  // Opcional FK
  check('scheduleId').optional({ nullable: true, checkFalsy: true }).isInt({ min: 1 }).toInt(),

  // Email válido
  check('email').optional({ nullable: true }).isEmail().trim(),

  // URL válida
  check('url').optional({ nullable: true }).isURL().trim(),

  // Booleano
  check('isDefault').optional().isBoolean().toBoolean(),

  // Validación custom (función auxiliar dada por el examen)
  check('startTime').exists().custom(validateTimeFormat),
  check('endTime').exists().custom(validateTimeFormat).custom(validateEndTimeAfterStartTime),

  // Validación custom async (comprobar relación en BD)
  check('scheduleId').optional({ nullable: true }).isInt({ min: 1 }).toInt()
    .custom(checkScheduleBelongsToRestaurant),

  // Campo que NO debe existir
  check('userId').not().exists().withMessage('userId cannot be provided')
]
```

### Cuándo usar `exists` vs `optional`
- `exists()` → campo **obligatorio** — si no viene: 422
- `optional()` → campo **opcional** — si no viene: no valida. Si viene: aplica las reglas

### Función custom async típica del examen
```javascript
const checkScheduleBelongsToRestaurant = async (value, { req }) => {
  try {
    const schedule = await Schedule.findByPk(value)
    if (!schedule) return Promise.reject(new Error('Schedule does not exist.'))
    if (schedule.restaurantId !== req.body.restaurantId) {
      return Promise.reject(new Error('Schedule does not belong to restaurant.'))
    }
    return Promise.resolve()
  } catch (err) {
    return Promise.reject(new Error(err))
  }
}
```

---

## 5. CONTROLADOR

### Concepto
Lógica de negocio. Solo se ejecuta si todos los middlewares pasaron. Genera el 200 o el 500.

### Los 5 métodos estándar
```javascript
import { Entidad, OtraEntidad } from '../models/models.js'

// INDEX — listar todos (filtrado por param)
const index = async function (req, res) {
  try {
    const items = await Entidad.findAll({
      where: { restaurantId: req.params.restaurantId }
    })
    res.json(items)
  } catch (err) { res.status(500).send(err) }
}

// SHOW — detalle con includes
const show = async function (req, res) {
  try {
    const item = await Entidad.findByPk(req.params.entidadId, {
      include: [{ model: OtraEntidad, as: 'otraEntidad' }]
    })
    res.json(item)
  } catch (err) { res.status(500).send(err) }
}

// CREATE — crear nuevo
const create = async function (req, res) {
  try {
    const newItem = Entidad.build(req.body)
    newItem.restaurantId = req.params.restaurantId  // asignar desde URL
    newItem.userId = req.user.id                    // asignar usuario autenticado
    const item = await newItem.save()
    res.json(item)
  } catch (err) { res.status(500).send(err) }
}

// UPDATE — actualizar
const update = async function (req, res) {
  try {
    await Entidad.update(req.body, {
      where: { id: req.params.entidadId }
    })
    const updated = await Entidad.findByPk(req.params.entidadId)
    res.json(updated)
  } catch (err) { res.status(500).send(err) }
}

// DESTROY — eliminar
const destroy = async function (req, res) {
  try {
    const destroyed = await Entidad.destroy({
      where: { id: req.params.entidadId }
    })
    res.json(destroyed === 1 ? 'Successfully deleted.' : 'Not found.')
  } catch (err) { res.status(500).send(err) }
}
```

### Casos especiales del examen

#### markDefault — marcar como predeterminado (ShippingAddress)
```javascript
// Patrón: desmarcar todos los del usuario, luego marcar el indicado
const markDefault = async function (req, res) {
  try {
    // Desmarcar todas las direcciones del usuario
    await ShippingAddress.update(
      { isDefault: false },
      { where: { userId: req.user.id } }
    )
    // Marcar la indicada
    await ShippingAddress.update(
      { isDefault: true },
      { where: { id: req.params.shippingAddressId } }
    )
    const updated = await ShippingAddress.findByPk(req.params.shippingAddressId)
    res.json(updated)
  } catch (err) { res.status(500).send(err) }
}
```

#### create con lógica de negocio (primera dirección → default)
```javascript
const create = async function (req, res) {
  try {
    const count = await ShippingAddress.count({ where: { userId: req.user.id } })
    const newAddress = ShippingAddress.build(req.body)
    newAddress.userId = req.user.id
    newAddress.isDefault = (count === 0) // primera → predeterminada
    const address = await newAddress.save()
    res.json(address)
  } catch (err) { res.status(500).send(err) }
}
```

#### Devolver 409 en controlador (conflicto de negocio)
```javascript
// Cuando la lógica de negocio impide la operación
return res.status(409).send('No puedes hacer esto porque...')
```

---

## 6. MIDDLEWARE PROPIO

### checkOwnership genérico
```javascript
// Patrón para cualquier entidad
export const checkShippingAddressOwnership = async (req, res, next) => {
  try {
    const address = await ShippingAddress.findByPk(req.params.shippingAddressId)
    if (address.userId !== req.user.id) {
      return res.status(403).send('Not enough privileges.')
    }
    return next()
  } catch (err) {
    return res.status(500).send(err)
  }
}
```

### checkCustomerHasNotReviewed (Reviews)
```javascript
const checkCustomerHasNotReviewed = async (req, res, next) => {
  try {
    const existing = await Review.findOne({
      where: {
        restaurantId: req.params.restaurantId,
        customerId: req.user.id
      }
    })
    if (existing) {
      return res.status(409).send('You already reviewed this restaurant.')
    }
    return next()
  } catch (err) {
    return res.status(500).send(err)
  }
}
```

### userHasPlacedOrderInRestaurant (Reviews)
```javascript
const userHasPlacedOrderInRestaurant = async (req, res, next) => {
  try {
    const order = await Order.findOne({
      where: {
        restaurantId: req.params.restaurantId,
        userId: req.user.id
      }
    })
    if (!order) {
      return res.status(409).send('You have not placed any order in this restaurant.')
    }
    return next()
  } catch (err) {
    return res.status(500).send(err)
  }
}
```

---

## 7. CONSULTAS ESPECIALES

### findAll con WHERE + includes (showWithActiveProducts)
```javascript
// Patrón: filtrar productos por horario activo en este momento
const currentTime = new Date().toTimeString().split(' ')[0] // HH:mm:ss

const restaurant = await Restaurant.findByPk(req.params.restaurantId, {
  attributes: { exclude: ['userId'] },
  include: [{
    model: Product,
    as: 'products',
    where: { scheduleId: { [Op.ne]: null } }, // excluir sin horario
    required: false,
    include: [{
      model: Schedule,
      as: 'schedule',
      where: {
        startTime: { [Op.lte]: currentTime }, // startTime <= ahora
        endTime: { [Op.gte]: currentTime }    // endTime >= ahora
      },
      required: true // INNER JOIN — excluye productos sin horario activo
    }]
  }]
})
```

### Operadores de Sequelize más usados
```javascript
import { Op } from 'sequelize'

{ [Op.ne]: null }    // != null
{ [Op.lte]: valor }  // <= valor
{ [Op.gte]: valor }  // >= valor
{ [Op.gt]: valor }   // > valor
{ [Op.lt]: valor }   // < valor
{ [Op.in]: [1,2,3] } // IN (1,2,3)
```

### include required: true vs false
- `required: true` → INNER JOIN → solo devuelve padres que tienen al menos un hijo que cumple la condición
- `required: false` → LEFT JOIN → devuelve todos los padres, con o sin hijos

---

## 8. ATRIBUTOS VIRTUALES

### Concepto
Campo calculado que no existe en la BD. Se calcula en tiempo de ejecución.

### Patrón completo (avgStars en Restaurant)
```javascript
// 1. Definir el campo virtual en init()
avgStars: {
  type: DataTypes.VIRTUAL,
  get () { return this.getDataValue('avgStars') },
  set (value) { this.setDataValue('avgStars', value) }
}

// 2. Implementar la función de cálculo
async getAvgStars () {
  try {
    const reviews = await this.getReviews() // usa la asociación hasMany
    if (!reviews || reviews.length === 0) return null
    const total = reviews.reduce((acc, r) => acc + r.stars, 0)
    return total / reviews.length
  } catch (err) {
    return null
  }
}

// 3. El hook afterFind ya está configurado en el modelo base del examen:
hooks: {
  afterFind: async (result) => {
    if (Array.isArray(result)) {
      await Promise.all(result.map(async r => {
        r.dataValues.avgStars = await r.getAvgStars()
      }))
    } else if (result) {
      result.dataValues.avgStars = await result.getAvgStars()
    }
  }
}
```

---

## 9. ERRORES COMUNES

| Error | Causa | Solución |
|-------|-------|----------|
| `Route.put() requires a callback function but got [object String]` | `handleFilesUpload(['logo'], process.env.FOLDER)` con la coma fuera del paréntesis | Meter ambos argumentos DENTRO del paréntesis |
| `Cannot find package 'express'` | No se hizo npm install en la carpeta correcta | `cd DeliverUS-Backend && npm install` |
| `ERR_MODULE_NOT_FOUND` | Import con ruta incorrecta | Revisar rutas relativas de imports |
| `auth_gssapi_client` en MariaDB | Método de autenticación incompatible | Recrear usuario con `mysql_native_password` |
| `Foreign key constraint is incorrectly formed` | Nombre de tabla referenciada mal escrito | Mayúsculas exactas: `'Restaurants'` no `'restaurants'` |
| `Unknown column 'X' in INSERT INTO` | Campo en seeder que no está en la migración | Añadir el campo a la migración |
| Test devuelve 500 en lugar de 422 | Validación de FK llega a la BD con valor inválido | Añadir `{ checkNull: true }` y `notEmpty()` |
| Test devuelve 200 en lugar de 422 | `optional()` en campo que debería ser obligatorio | Cambiar a `exists({ checkFalsy: true })` |

---

## 10. CÓDIGO RÁPIDO DE REFERENCIA

### Proceso mental al recibir el examen
```
1. Leer el README → identificar la nueva entidad y sus campos
2. Leer el diagrama → ver relaciones (FK, 1:N, N:M)
3. Leer los tests → ver qué rutas, qué códigos HTTP, qué validaciones
4. Completar en orden: migración → modelo → rutas → validación → controlador
```

### De dónde saco cada cosa
| ¿Qué necesito saber? | ¿Dónde lo encuentro? |
|----------------------|----------------------|
| Nombres de campos | Diagrama + README (ejemplo JSON) + Migración dada |
| Campos obligatorios/opcionales | Migración (allowNull) + Tests (pruebas de 422) |
| Nombres de archivos a importar | Carpeta `src/middlewares/`, `src/controllers/validation/` |
| Variables de entorno | `.env.example` |
| Funciones auxiliares | El propio archivo de validación dado |
| Qué middlewares usar | README (lista de códigos de error) + Tests |

### Import rápido de lo que siempre necesitas
```javascript
// En rutas
import { isLoggedIn, hasRole } from '../middlewares/AuthMiddleware.js'
import { handleValidation } from '../middlewares/ValidationHandlingMiddleware.js'
import { checkEntityExists } from '../middlewares/EntityMiddleware.js'

// En controladores
import { Op } from 'sequelize'

// En validación
import { check } from 'express-validator'
```

### Tabla rápida de códigos HTTP
| Código | Cuándo | Lo genera |
|--------|--------|-----------|
| 200 | Éxito | Controlador (`res.json()`) |
| 401 | No autenticado | `isLoggedIn` |
| 403 | Sin permisos | `hasRole`, `checkOwnership` |
| 404 | No existe | `checkEntityExists` |
| 409 | Conflicto de negocio | Middleware propio o controlador |
| 422 | Datos inválidos | `handleValidation` |
| 500 | Error inesperado | `res.status(500).send(err)` |

---

## 🎯 ESTO SÍ CAE EN EL EXAMEN

### Patrón de los 3 exámenes analizados

| | Schedules | ShippingAddress | Reviews |
|--|-----------|-----------------|---------|
| **Nueva entidad** | Schedule | ShippingAddress | Review |
| **Relación** | Schedule → Restaurant (N:1) | ShippingAddress → User (N:1) | Review → Restaurant (N:1), Review → User (N:1) |
| **Modifica entidad existente** | Product (añade scheduleId) | Order (añade shippingAddressId) | Restaurant (añade avgStars) |
| **Validación especial** | formato HH:mm:ss, endTime > startTime | campos obligatorios (alias, street, city...) | stars entre 0 y 5 |
| **Middleware propio** | No | checkShippingAddressOwnership | checkReviewOwnership, userHasPlacedOrder, checkNotReviewed |
| **Consulta especial** | showWithActiveProducts (filtrar por hora) | No | avgStars (calcular media) |
| **Lógica de negocio en controller** | No | primera dirección → isDefault=true | 409 si ya revisó |

### Lo que siempre piden:
1. ✅ Migración de la nueva entidad (campos + FK + timestamps)
2. ✅ Modificar migración existente (añadir FK a tabla ya creada)
3. ✅ Modelo con `associate()` y campos
4. ✅ Modificar modelo existente (añadir relación inversa)
5. ✅ Rutas con middlewares correctos
6. ✅ Validaciones con express-validator
7. ✅ CRUD en el controlador
8. ✅ Algún middleware propio de ownership o negocio
9. ✅ Alguna consulta especial (filtrado, cálculo, virtual)
