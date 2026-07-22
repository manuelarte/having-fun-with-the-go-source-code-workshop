# Divirtiéndose con el Código Fuente de Go

¡Bienvenido a un taller interactivo donde aprenderás a modificar y experimentar con el código fuente del lenguaje de programación Go! Este taller práctico te guiará para entender, compilar y hacer cambios en el compilador y el runtime de Go.

**Este taller usa Go 1.26.1** - haremos checkout de ese release tag concreto para asegurar la consistencia en todos los ejercicios.

## Languages / 语言

- English: [README.md](./README.md) · exercises as `*.md`
- Español: [README.es.md](./README.es.md) · exercises as `*.es.md`
- 中文: [README.zh.md](./README.zh.md) · exercises as `*.zh.md`

## Requisitos Previos

- Conocimientos básicos de programación en Go
- Familiaridad con herramientas de línea de comandos
- Git instalado en tu sistema
- **Compilador de Go 1.24 o superior** (necesario para el proceso de bootstrap de la compilación)
- Al menos 4GB de espacio libre en disco

## Descripción del Taller

Este taller consta de 12 ejercicios que te llevarán por el proceso, desde compilar Go desde el código fuente hasta hacer modificaciones en distintos lugares del compilador, las herramientas y el runtime. Obtendrás algunas nociones sobre los internos de Go, desde cosas como el lexer o el parser hasta comportamientos del runtime:

### [Ejercicio 0: Introducción y Configuración](./exercises/00-introduction-setup.es.md)

Empieza clonando y configurando el entorno del código fuente de Go.

### [Ejercicio 1: Compilando Go Sin Cambios](./exercises/01-compile-go-unchanged.es.md)

Aprende a compilar el toolchain de Go desde el código fuente sin modificaciones.

### [Ejercicio 2: Añadiendo el Operador Flecha "=>" para Goroutines](./exercises/02-scanner-arrow-operator.es.md)

Aprende a modificar el scanner/lexer añadiendo "=>" como sintaxis alternativa para iniciar goroutines.

### [Ejercicio 3: Múltiples Keywords "go" - Mejora del Parser](./exercises/03-parser-multiple-go.es.md)

Aprende a modificar el parser permitiendo múltiples keywords "go" consecutivos (go go go myFunction).

### [Ejercicio 4: Parámetros de Inlining - Experimentos con Function Inlining](./exercises/04-compiler-inlining-parameters.es.md)

Explora el comportamiento del inliner modificando los parámetros de inlining de funciones.

### [Ejercicio 5: Modificación de gofmt - Indentación y Transformación AST](./exercises/05-gofmt-ast-transformation.es.md)

Modifica gofmt para usar 4 espacios en lugar de tabs y añade una transformación AST personalizada que reemplaza "hello" con "helo".

### [Ejercicio 6: Pase SSA - Detectando División por Potencias de Dos](./exercises/06-ssa-power-of-two-detector.es.md)

Crea un pase SSA personalizado en el compilador que detecta operaciones de división por potencias de dos que podrían optimizarse con bit shifts.

### [Ejercicio 7: Go Paciente - Haciendo que Go Espere a las Goroutines](./exercises/07-runtime-patient-go.es.md)

Modifica el runtime de Go para esperar a que todas las goroutines terminen antes de finalizar el programa.

### [Ejercicio 8: Detective de Goroutines Dormidas - Monitoreo del Estado del Runtime](./exercises/08-goroutine-sleep-detective.es.md)

Añade logging al scheduler de Go para monitorear las goroutines que se van a dormir.

### [Ejercicio 9: Select Predecible - Eliminando la Aleatoriedad del Select de Go](./exercises/09-predictable-select.es.md)

Modifica la implementación del select de Go para que sea determinista en lugar de aleatoria.

### [Ejercicio 10: Stack Traces Estilo Java - Haciendo los Panics de Go Familiares](./exercises/10-java-style-stack-traces.es.md)

Transforma los verbosos stack traces de Go al formato estilo Java.

### [Ejercicio 11: D&D Work Stealing - Tirando Dados por Goroutines](./exercises/11-dnd-work-stealing.es.md)

Añade una tirada de dado al algoritmo de work stealing del scheduler: los P deben sacar más de 10 en un d20 para robar goroutines.

## Cómo Empezar

1. Empieza con el [Ejercicio 0](./exercises/00-introduction-setup.es.md) para configurar tu entorno
2. Ve resolviendo los ejercicios en orden
3. Después del ejercicio 1, puedes elegir el ejercicio que quieras.

## Estructura del Repositorio

```
.
├── README.md                 # Este archivo (inglés)
├── README.es.md              # Versión en español
├── README.zh.md              # Versión en chino
├── exercises/               # Archivos individuales de ejercicios (markdown)
│   ├── 00-introduction-setup.md
│   ├── 01-compile-go-unchanged.md
│   ├── 02-scanner-arrow-operator.md
│   └── ...
├── website-generator/       # Programa en Go para generar el sitio web desde markdown
│   ├── main.go
│   ├── templates.go
│   └── README.md
├── website/                 # Sitio web generado (HTML)
│   ├── index.html
│   ├── 00-introduction-setup.html
│   └── ...
├── Makefile                 # Automatización de la compilación
└── go/                      # Código fuente de Go (clonado durante la configuración)
```

## Generador del Sitio Web

Este repositorio incluye un programa en Go que genera automáticamente un sitio web estático a partir de los archivos markdown de los ejercicios.

### Generar el Sitio Web

```bash
# Usando make (recomendado)
make website

# O ejecutándolo directamente
cd website-generator
go run . -exercises ../exercises -output ../website
```

### Servir Localmente

```bash
# Inicia un servidor web local
make serve

# Luego abre http://localhost:8000 en tu navegador
```

El generador del sitio web:
- Convierte markdown a HTML usando [blackfriday](https://github.com/russross/blackfriday)
- Preserva todo el formato, los emojis y los bloques de código
- Genera la navegación entre ejercicios
- Crea una página índice con la descripción de los ejercicios
- Incluye estilos CSS responsivos

Consulta [website-generator/README.md](website-generator/README.md) para más detalles.

## Consejos para Tener Éxito

- Tómate tu tiempo con cada ejercicio: ¡los internos del compilador son complejos!
- No dudes en explorar el código fuente de Go más allá de lo estrictamente necesario
- Usa `git` para llevar el control de tus cambios y revertirlos cuando lo necesites
- Prueba tus modificaciones a fondo con distintos programas de Go

## Recursos

- [Visión General del Compilador de Go](https://github.com/golang/go/tree/master/src/cmd/compile)
- [Especificación del Lenguaje Go](https://go.dev/ref/spec)
- [Documentación del Runtime de Go](https://pkg.go.dev/runtime)

### Referencias en Vídeo

Estos ejercicios del taller se basan en ideas de mis charlas:

- [Understanding the Go Compiler](https://www.youtube.com/watch?v=qnmoAA0WRgE) - Una inmersión profunda en el proceso de compilación de Go
- [Understanding the Go Runtime](https://www.youtube.com/watch?v=YpRNFNFaLGY) - Una exploración del sistema de runtime de Go

## Al Completar el Taller

Cuando termines todos los ejercicios, habrás:

- **Compilado Go desde el código fuente** y entendido el proceso de bootstrap
- **Modificado la sintaxis del lenguaje** cambiando el comportamiento del scanner y el parser
- **Personalizado herramientas de desarrollo** como gofmt y las optimizaciones del compilador
- **Implementado optimizaciones SSA** en el backend del compilador
- **Modificado el comportamiento del runtime**, incluyendo los puntos de entrada del programa y el monitoreo del scheduler
- **Alterado algoritmos de concurrencia** como la aleatoriedad de la sentencia select
- **Personalizado el reporte de errores** con un formato de stack trace estilo Java

**¡Enhorabuena!** Habrás ganado la confianza para seguir explorando el código fuente de Go. Estos conocimientos te permiten:

- Empezar a hacer pequeñas contribuciones al proyecto Go
- Construir variantes y herramientas personalizadas del lenguaje
- Entender algunas de las decisiones de diseño del lenguaje y del runtime

## Contribuir

¿Has encontrado un problema, tienes una idea de mejora o quieres añadir más ejercicios? ¡Por favor [abre un issue](https://github.com/jespino/having-fun-with-the-go-source-code-workshop/issues) o envía un pull request!

## Licencia

Este proyecto está licenciado bajo la Licencia MIT - consulta el archivo [LICENSE](LICENSE) para más detalles.

---

**¡Feliz programación y bienvenido al mundo de los internos de Go!**
