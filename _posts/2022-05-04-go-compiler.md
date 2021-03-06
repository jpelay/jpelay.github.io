## Agregando bucles do-while al compilador de Go

Hace un tiempo me encontré con este [magnífico post](https://eli.thegreenplace.net/2019/go-compiler-internals-adding-a-new-statement-to-go-part-1/) de Eli Benderksy el cual trata acerca de agregar un nuevo tipo de sentencia a Go. Este es un lenguaje que casi no conozco, así que meterme de lleno en el compilador me pareció una excelente idea para conocer un poco más del lenguaje y de cómo funciona un compilador usado en producción. Sin embargo, el post de Eli es viejo y varios de los pasos listados en él se encuentran desfasados, así que este es mi intento de retomar su trabajo y dar los pasos actuales de cómo funciona el compilador y explicar un poco acerca de su estructura.

### Lo que haremos
Go es un lenguaje minimalista y solamente tiene una palabra reservada para señalar un bucle, el cual puede servir de varias formas dependiendo de sus componentes, los cuales son opcionales, pudiendo hacer bucles `for`, `while` e infinitos usando solo una palabra clave:

```go
for Inic; Cond; Post {
    Cuerpo
}
```

Sin embargo, para poder hacer un bucle `do-while` habría que hacer algo como lo siguiente:

```go
i := 0
for {
	i++
	if i >= 5 {
		break
	}
}
```

Que compila a:

```text
	XORL AX, AX
main_pc2:
	INCQ AX
	CMPQ AX, $5
	JLT main_pc2
	RET
```

Lo que podemos ver en este código es que Go, incluso no teniendo una sintaxis dedicada a los bucles `do-while` sí sabe interpretar nuestras intenciones cuando hacemos uno. Como vemos el código del bucle externo fue removido, no hay un salto a la condición, pasa directo al cuerpo el cual se garantiza que se ejecute al menos una vez.

Nuestra tarea es emular este comportamiento añadiendo los bucles `do-while` directamente, y siguiendo todo el proceso del compilador hasta la genración del código máquina para emular ese comportamiento. Un bucle de este tipo luciría de la siguiente manera:

``` go
i := 0
do {
	i++
} while i <= 5;
```

### La estructura del compilador
Todo el código del compilador vive en la carpeta `src/cmd/compile/internal`, la cual está dividida en paquetes a los que iré haciendo referencia a lo largo del post. El código de Go puede ser clonado fácilmente desde [este mirror en github](https://github.com/golang/go/).

### El analizador léxico
Lo primero que tenemos que hacer es agregar los tokens que el analizador léxico tomará como nuestros bucles `do-while`. Go tiene una forma más bien extraña de definir sus tokens, los cuales se encuentran en el archivo `syntax/tokens.go` el cual contiene un enumerado con los tokens.

```go
_Default // default
_Defer // defer
_Do // do
_While // while
_Else // else
_Fallthrough // fallthrough
```

Para que esta definición sea agregada al compilador debemos instalar la herramienta stringer de Go, y usar este comando dentro la carpeta syntax:

```
go generate tokens.go
```

Este comando va a generar el archivo `tokens_string.go`, que si revisamos podemos notar que contiene dos nuevas entradas en el arreglo de tokens, una para `do` y otra para `while`. Ahora podemos tratar de compilar Go, para esto vamos a la carpeta `src` y corremos el comando `./make.bash`. Si tratamos de compilar ahora el compilador se queja:

```
/home/capybara/repos/go/src/hash/crc32/crc32_amd64.go:217:3: syntax error: unexpected do, expecting }
/home/capybara/repos/go/src/hash/crc32/crc32_amd64.go:218:29: syntax error: unexpected do, expecting expression
/home/capybara/repos/go/src/hash/crc32/crc32_amd64.go:219:9: syntax error: unexpected do, expecting expression
/home/capybara/repos/go/src/hash/crc32/crc32_amd64.go:221:2: syntax error: non-declaration statement outside function body  

go tool dist: FAILED: /home/capybara/repos/go/pkg/tool/linux_amd64/compile -std -pack -o /tmp/go-tool-dist-976817820/hash/crc32/_go_.a -p hash/crc32 -importcfg /tmp/go-tool-dist-976817820/hash/crc32/importcfg -asmhdr /tmp/go-tool-dist-976817820/hash/crc32/go_asm.h -symabis /tmp/go-tool-dist-976817820/hash/crc32/symabis /home/capybara/repos/go/src/hash/crc32/crc32.go /home/capybara/repos/go/src/hash/crc32/crc32_amd64.go /home/capybara/repos/go/src/hash/crc32/crc32_generic.go: exit status 2
```

Ups, parece que en varios sitios dentro del código fuente de Go usan la palabra `do` como nombre para variables. Como en la línea que nos indica el compilador, en el modulo hash:

```go
if len(p) >= 64 {
	left := len(p) & 15
	do := len(p) - left
	crc = ^ieeeCLMUL(^crc, p[:do])
	p = p[do:]
}
```

Para solucionar esto cambiamos el token de `do` a `do_` usamos el comando `go generate` e intentamos compilar nuevamente y ¡funciona! Por alguna razón Eli tuvo problemas con el hash de los tokens, pero es problable que haya tenido suerte con los tokens que agregué ya que su hash no choca con el de las otras keywords de Go.

### Análisis sintáctico
Luego de convertir el código fuente en una cadena de tokens, el compilador hace el análisis sintáctico. Go usa un análizador sintáctico de descenso recursivo bastante estándar. El parser (como también son conocidos los analizadores sintácticos), va a leer la cadena de tokens y la convertirá en un árbol de sintaxis concreta (CST), que luego será manipulado en las siguientes fases de compilación y convertido en un AST. Este paso de un CST a un AST quedó de los primeros días del compilador cuando estaba escrito en C y luego fue convertido a Go.

Lo siguiente que debemos hacer es agregar el nodo del CST correspondiente a nuestro bucle do-while:

```go
DoWhileStmt struct {
	Cond Expr
	Body *BlockStmt
	stmt
}	
```
 
Una sentencia do while luciria como:

```go
do_ {
	<body>
} <cond>;

```

Para hacer que Go parsee la sentencia debemos ir hacia el archivo `syntax/parser.go` y agregar un nuevo caso en el `switch` de los tokens:

``` go
case _Do:
	return p.doWhileStmt()
```

Y ahora sí agregamos la función encargada de parsear la sentencia:

```go
func (p *parser) doWhileStmt() Stmt {
	if trace {
		defer p.trace("doWhileStmt")()
	}
	
	s := new(DoWhileStmt)
	s.pos = p.pos()
	// advancing the _Do token
	p.next()
	s.Body = p.blockStmt("dowhile clause")
	_, s.Cond, _ = p.header(_While)
	return s
}
```

La función en sí es bastante sencilla, primero necesitamos avanzar el token `_Do` para que la función `blockStmt` parsee el cuerpo del bucle. Luego reutilizamos la función `header` que es usada por los bucles `for` y los `if`, y la modificamos un poco para que sea capaz de procesar los bucles `do-while`. En realidad solo necesitamos agregar un pequeño condicional para que no entrar en los casos extra de sentencias de inicialización y posteriores que sí usan los `for` y los `if`:

```go
if p.tok != _Semi {
	// ...
	init = p.simpleStmt(nil, keyword)
	// ...
}
var condStmt SimpleStmt
var semi struct {
	pos Pos
	lit string // valid if pos.IsKnown()
}

if p.tok != _Lbrace && keyword != _While {
	// ...	
} else {
	condStmt = init
	init = nil
}
// ...

```

Para poder imprimir el CST podemos agregar unos print de debug en la función `LoadPackage` del paquete `noder`. Esta función es la que se encarga de invocar al parser:

```go
p.file, _ = syntax.Parse(fbase, f, p.error, p.pragma, mode) // errors are tracked via p.error
if len(os.Getenv("FOO")) > 0 {
	fmt.Printf("FOO='%s'\n", os.Getenv("FOO"))
	fmt.Println("Dumping", p.file.PkgName)
	syntax.Fdump(os.Stdout, p.file)
}
```

Si queremos mostrar el CST de este programa:

```go
package main

func main() {
	i := 1
	do_ {
		i++
	} while i < 10;
}
```

Lo compilamos usando este comando:

```
FOO=FOO <raiz de la carpeta del proyecto>/go/bin run dowhile.go
```

Y ¡éxito! Ya podemos parsear bucles do-while:

```text
    25  .  .  .  .  .  1: *syntax.DoWhileStmt {
    26  .  .  .  .  .  .  Cond: *syntax.Operation {
    27  .  .  .  .  .  .  .  Op: <
    28  .  .  .  .  .  .  .  X: i @ ./dowhile.go:7:10
    29  .  .  .  .  .  .  .  Y: *syntax.BasicLit {
    30  .  .  .  .  .  .  .  .  Value: "10"
    31  .  .  .  .  .  .  .  .  Kind: 0
    32  .  .  .  .  .  .  .  .  Bad: false
    33  .  .  .  .  .  .  .  }
    34  .  .  .  .  .  .  }
    35  .  .  .  .  .  .  Body: *syntax.BlockStmt {
    36  .  .  .  .  .  .  .  List: []syntax.Stmt (1 entries) {
    37  .  .  .  .  .  .  .  .  0: *syntax.AssignStmt {
    38  .  .  .  .  .  .  .  .  .  Op: +
    39  .  .  .  .  .  .  .  .  .  Lhs: i @ ./dowhile.go:6:3
    40  .  .  .  .  .  .  .  .  .  Rhs: nil
    41  .  .  .  .  .  .  .  .  }
    42  .  .  .  .  .  .  .  }
    43  .  .  .  .  .  .  .  Rbrace: syntax.Pos {}
    44  .  .  .  .  .  .  }
```


### Chequeo de tipos

La siguiente fase del compilador es hacer un chequeo de los tipos. Esta fase se encarga de que los tipos usados en las sentencias sean correctos, por ejemplo, que el dato asignado a una variable sea el mismo de la variable (o [compatible](https://go.dev/ref/spec#Assignability)), que la variable condicional de los `if` sean de tipo booleano, etc. El chequeo de tipos de Go también hace algunas cosas extras, como apunta [Eli](https://eli.thegreenplace.net/2019/go-compiler-internals-adding-a-new-statement-to-go-part-1/): enlazar los identificadores a sus declaraciones, calcular las constantes de tiempo de compilación, inferencia de tipos, etc.

La fase de chequeo de tipos se hace justo antes de la generación del AST. La función que se encarga de llamar al chequeador de tipos se encuentra en el paquete `noder` en el archivo `irgen.go`:

```go
// check2 type checks a Go package using types2, and then generates IR
// using the results.
func check2(noders []*noder) {
	m, pkg, info := checkFiles(noders)

	g := irgen{
		target: typecheck.Target,
		self: pkg,
		info: info,
		posMap: m,
		objs: make(map[types2.Object]*ir.Name),
		typs: make(map[types2.Type]*types.Type),
	}
	g.generate(noders)
}
```

Al hacerse antes de la transformación al AST el chequeo de tipos en Go funciona sobre el CST que se generó en la sección de análisis sintáctico. Para agregar el caso del chequeo de tipos para el bucle `do-while` seguimos el caso del bucle `for` y agregamos este código en el archivo `stmt.go` del paquete `types2`:

```go
case *syntax.DoWhileStmt:
	inner |= breakOk | continueOk
	check.openScope(s, "do_")
	defer check.closeScope()
	
	if s.Cond != nil {
		var x operand
		check.expr(&x, s.Cond)
		if x.mode != invalid && !allBoolean(x.typ) {
			check.error(s.Cond, "non-boolean condition in do-while statement")
			}
	}  
	
	check.stmt(inner, s.Body)  
}
```

Primero debemos comprobar que la expresión que se encuentra en el condicional del bucle sea de tipo booleano, y luego llamamos a la función que va a comprobar el cuerpo del bucle. No tenemos mucho trabajo acá, puesto que nuestro bucle `do-while` es bastante simple.

Ahora, si buscas en el código del compilador de Go por otras funciones que hagan chequeo de tipos, econtrarás que existe un paquete llamado `typechek` que actúa sobre el AST y también hace un chequeo de tipos. Las funciones en este paquete son muy similares a las del paquete `types2`. Mi primera teoría era que el compilador primero chequeaba el CST antes de optimizaciones, y de nuevo luego de aplicar las optimizaciones. Pero estaba equivocado. En realidad este paquete es llamado cuando el compilador genera código durante su ejecución, el cual luego debe ser chequeado (aún así puedo estar equivocado respecto a esto, si tienes algún comentario puedes agregar un issue en el [repositorio de este blog](github.com/jpelay/jpelay.github.io/) o enviarme un correo a pelayjesus@gmail.com).

Para propósitos ilustrativos, aquí dejo como quedaría el caso del bucle `do-while` en este paquete:

```go
// tcDoWhile typechecks an ODOWHILE node.
func tcDoWhile(n *ir.DoWhileStmt) ir.Node {
	n.Cond = Expr(n.Cond)
	n.Cond = DefaultLit(n.Cond, nil)
	if n.Cond != nil {
		t := n.Cond.Type()
		if t != nil && !t.IsBoolean() {
			base.Errorf("non-bool %L used as do-while condition", n.Cond)
		}
	}
	Stmts(n.Body)
	return n
}
```

### Creando el árbol de sintaxis abstracta
La siguiente fase de compilación involucra transformar el árbol generado por el paquete `syntax` a otra representación que es entendida por las étapas posteriores, que involucran las optimizaciones y la generación de código. Según los autores de Go este paso puede ser refactorizado en el futuro, pero aún es necesario.

La definición de los nodos del árbol de sintaxis abstracta (AST) se encuentran en el paquete `ir`, significa *intermediate representation* o representación intermedia. Anteriormente el compilador usaba un tipo de nodo genérico, pero posterioremente crearon una interfaz génerica y le dieron a cada constructo de la gramática su propio nodo. Es por esto que no podemos hacer una traducción 1:1 del post de Eli, la versión que él maneja en ese post es antigua y a partir de aquí las divergencias se hacen notables.

Lo primero que debemos hacer es añadir el código del operador que servirá para identificar nuestros nodos. Estos códigos se encuentran definidos en `ir/node.go`:

```go
OFORUNTIL
ODOWHILE // do_ { Body } while Cond;
OGOTO // goto Label
OIF // if Init; Cond { Then } else { Else }
OLABEL // Label:
OGO // go Call
```

Estos códigos de operación funcionan de manera similar a los tokens que vimos anteriormente, así que de nuevo debemos ejecutar el comando stringer dentro de la carpeta `ir`:

```bash
go generate node.go
```

Este comando va a generar el archivo `op_string.go` que contiene un arreglo con el nombre de los operadores.

Luego de agregar el tipo de operador, podemos agregar, ahora sí, nuestra definición del bucle. Aquí podemos tomar prestada la definición que usa el bucle `for` y quitar las partes que no nos interesan. Además debemos definir dos operaciones que deben definir todos los nodos. Esta definición la podemos agregar en el archivo `ir/stmt.go`: 

```go
// A DoWhile is loop with the structure do_ { Body } while Cond;
// where the Body is guaranteed to be executed at least once
type DoWhileStmt struct {
	miniStmt
	Label *types.Sym
	Cond Node
	Body Nodes
	HasBreak bool
}

func NewDoWhileStmt(pos src.XPos, cond Node, body []Node) *DoWhileStmt {
	n := &DoWhileStmt{Cond: cond}
	n.pos = pos
	n.op = ODOWHILE
	n.Body = body
	return n
}
 
func (n *DoWhileStmt) SetOp(op Op) {
	if op != ODOWHILE {
		panic(n.no("SetOp " + op.String()))
	}
	n.op = op
}
```

Al principio del archivo hay un `switch` que decide que tipo de nodo de AST crear dependiendo del nodo del CST que esté recorriendo en ese momento, así que lo agregamos:

```go
case *syntax.IfStmt:
	return g.ifStmt(stmt)
case *syntax.ForStmt:
	return g.forStmt(stmt)
case *syntax.DoWhileStmt:
	return g.doWhileStmt(stmt)
```

Y también agregamos la función que transformará uno en el otro:

```go
func (g *irgen) doWhileStmt(stmt *syntax.DoWhileStmt) ir.Node {
	return ir.NewDoWhileStmt(g.pos(stmt), g.expr(stmt.Cond), g.blockStmt(stmt.Body))
}
```

Si tratamos de compilar esto, nos genera este error. 

```
/home/capybara/repos/go/src/cmd/compile/internal/typecheck/stmt.go:282: cannot use n (variable of type *ir.DoWhileStmt) as type ir.Node in return statement:
	*ir.DoWhileStmt does not implement ir.Node (missing Format method)
/home/capybara/repos/go/src/cmd/compile/internal/typecheck/typecheck.go:787: impossible type assertion: n.(*ir.DoWhileStmt)
	*ir.DoWhileStmt does not implement ir.Node (missing Format method)
go tool dist: FAILED: /usr/local/go/bin/go install -gcflags=-l -tags=math_big_pure_go compiler_bootstrap bootstrap/cmd/...: exit status 2
```

Esto es un paso común a partir de este momento. Compilar, obtener el error, ir hacia el sitio que nos apunta y hacer una búsqueda en el código a ver qué nos falta. Este proceso de familiarizarme con una base de código extraña considero fue la parte más provechosa de este proyecto. Este error en particular nos dice que no estamos implementando un método de la interfaz `ir.Node`. Estas definiciones se encuentran en el archivo `ir/node_gen.go`. Este archivo es generado automáticamente por el archivo `ir/mknode.go` de la siguiente forma:

```bash
go run mknode.go
```

Pero me retorna un error con los paquetes y no pude instalarlos. Sin embargo, al ser un archivo que sólo genera código, lo podemos agregar manualmente al archivo que este programa genera: `ir/node_gen.go`:

```go
func (n *DoWhileStmt) Format(s fmt.State, verb rune) { fmtNode(n, s, verb) }
func (n *DoWhileStmt) copy() Node {
	c := *n
	// c.init = copyNodes(c.init)
	c.Body = copyNodes(c.Body)
	return &c
}
func (n *DoWhileStmt) doChildren(do func(Node) bool) bool {
	if n.Cond != nil && do(n.Cond) {
		return true
	}
	if doNodes(n.Body, do) {
		return true
	}
	return false
}
func (n *DoWhileStmt) editChildren(edit func(Node) Node) {
	if n.Cond != nil {
		n.Cond = edit(n.Cond).(Node)
	}
	editNodes(n.Body, edit)
}
```

El propósito de `mknode.go` era precisamente no hacer eso, pero al ser esto un ejercicio de una sola vez no hay problema.

Luego de la generación del AST hay una fase que extra que vuelve recorrer el árbol mediante un recorrido preorden con el fin de chequear de nuevo por cualquier inconsistencia en el checado de tipos. Sólo debemos agregar el caso que recorre este nodo del árbol en el archivo syntax/walk.go:

```go
case *DoWhileStmt:	
	if n.Cond != nil {
		w.node(n.Cond)
	}
	w.node(n.Body)
```

### Análisis y reesccritura del AST
Luego de que el AST ha sido construido, el compilador se encarga de optimizarlo, transformarlo y analizarlo. Esta es la última fase del *frontend* del compilador. 

#### Análisis de escape
El análisis de escape es el proceso encargado de analizar donde las variables se deben alojar, si en la pila (*stack*) o en el montículo (*heap*). Para que el compilador pueda ejecutar el análisis de escape en nuestra sentencia, debemos agregar el caso en el archivo `escape/stmt.go`, de nuevo, guíandonos del caso del bucle `for`:

```go
case ir.ODOWHILE:
	n := n.(*ir.DoWhileStmt)
	e.loopDepth++
	e.discard(n.Cond)
	e.block(n.Body)
	e.loopDepth--
```

#### Reordenamiento del AST
Luego del análisis de escape, Go se encarga de separar las sentencias con el fin de hacer cumplir el orden de evaluación. Esto hace que luego reescribir el árbol sea más sencillo porque puede reordenar lo que él quiera dentro de una expresión. Un ejemplo sería reescribir `f[i] /= 2` en `f[i] = f[i] / 2` o dividir asignaciones múltiples en varias asignaciones simples, para lo cual puede introducir variables temporales, etc.

Este reordenamiento funciona por sentencia, así que debemos agregar la función que camina un nodo `do-while` en el archivo `walk/order.go

```go
//walkDoWhile walks an ODOWHILE node.
func walkDoWhile(n *ir.DoWhileStmt) ir.Node {
	walkStmtList(n.Body)
	if n.Cond != nil {
		init := ir.TakeInit(n.Cond)
		walkStmtList(init)
		n.Cond = walkExpr(n.Cond, &init)
		n.Cond = ir.InitExpr(init, n.Cond)
	}
	return n
}
```

También agregamos un nuevo caso en el `switch` del principio del archivo:

```go
case ir.ODOWHILE:
	n := n.(*ir.DoWhileStmt)
	return walkDoWhile(n)
```

#### Reescritura del AST
Una de las últimas fases del frontend del compilador antes de inicar con las fases del backend como la transformación a SSA. Esta fase se encarga de transformar el AST para hacer más sencilla su transformación a SSA. Por ejemplo, una de sus transformaciones incluye las sentencias `range`, las cuales son convertidas en formas más simples. Otro ejemplo es la reescritura de la función `append` de los `slices`, con el fin de detectar detectar cualquier efecto secundario antes de hacer la adición a la slice.

Para que el compilador recorra nuestro nodo en esta fase, agregamos otro caso en el switch del archivo `walk/stmt.go` y añadimos la función correspondiente:

```go
case ir.ODOWHILE:
	n := n.(*ir.DoWhileStmt)
	return walkDoWhile(n)
```

```go
//walkDoWhile walks an ODOWHILE node.
func walkDoWhile(n *ir.DoWhileStmt) ir.Node {
	walkStmtList(n.Body)
	if n.Cond != nil {
		init := ir.TakeInit(n.Cond)
		walkStmtList(init)
		n.Cond = walkExpr(n.Cond, &init)
		n.Cond = ir.InitExpr(init, n.Cond)
	}
	return n
}
```

Ahora estamos listos para generar el AST el programa que usamos en la sección de análisis sintáctico:

```text
. DCL # dowhile.go:4:2
. . NAME-main.i esc(no) Class:PAUTO Offset:0 OnStack Used int tc(1) # dowhile.go:4:2
. AS Def tc(1) # dowhile.go:4:4
. . NAME-main.i esc(no) Class:PAUTO Offset:0 OnStack Used int tc(1) # dowhile.go:4:2
. . LITERAL-1 int tc(1) # dowhile.go:4:7
. DOWHILE # dowhile.go:5:2
. DOWHILE-Cond
. . LT bool tc(1) # dowhile.go:7:12
. . . NAME-main.i esc(no) Class:PAUTO Offset:0 OnStack Used int tc(1) # dowhile.go:4:2
. . . LITERAL-10 int tc(1) # dowhile.go:7:14
. DOWHILE-Body
. . AS tc(1) # dowhile.go:6:4
. . . NAME-main.i esc(no) Class:PAUTO Offset:0 OnStack Used int tc(1) # dowhile.go:4:2
. . . ADD int tc(1) # dowhile.go:6:4
. . . . NAME-main.i esc(no) Class:PAUTO Offset:0 OnStack Used int tc(1) # dowhile.go:4:2
. . . . LITERAL-1 int tc(1) # dowhile.go:6:4
```

### Generación de SSA
Antes de generar el código máquina, Go primero genera algo llamado [SSA](https://en.wikipedia.org/wiki/Static_single_assignment_form), la cuál constituye la primera fase de del *backend* del compilador. En resumen el SSA es una forma de reprentación intermedia en donde cada variable es asignada sólo una vez. La ventaja del SSA es que permite aplicar optimizaciones mucho más facilmente, tales como eliminación de código muerto, reordenamiento de bucles, eliminación de comprabaciones de nil. El SSA luego es transformado en el código ensamblador de cada arquitectura, esto simplifica de soportar la compilación a distintas arquitecturas. Un resumen acerca del SSA puedes encontrarlo en el [README.md](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/README.md) del paquete `ssa` si quieres más detalles acerca de esta representación.

La generación de SSA de Go funciona teniendo bloques conectados entre sí formando un grafo de control de flujo. Para generar el SSA de nuestra sentencia `do-while` debemos ir hacia el archivo `ssa.go` del paquete `ssagen` y agregar el siguiente código en el `switch` dentro de la función `stmt`:

```go
case ir.ODOWHILE:
		n := n.(*ir.DoWhileStmt)
		bCond := s.f.NewBlock(ssa.BlockPlain)
		bBody := s.f.NewBlock(ssa.BlockPlain)
		bEnd := s.f.NewBlock(ssa.BlockPlain)

		bBody.Pos = n.Pos()

		// First jump to body
		b := s.endBlock()
		b.AddEdgeTo(bBody)

		// set up for continue/break in body
		prevContinue := s.continueTo
		prevBreak := s.breakTo
		s.continueTo = bCond
		s.breakTo = bEnd
		var lab *ssaLabel
		if sym := n.Label; sym != nil {
			// labeled for loop
			lab = s.label(sym)
			lab.continueTarget = bCond
			lab.breakTarget = bEnd
		}

		// generate body
		s.startBlock(bBody)
		s.stmtList(n.Body)

		// tear down continue/break
		s.continueTo = prevContinue
		s.breakTo = prevBreak
		if lab != nil {
			lab.continueTarget = nil
			lab.breakTarget = nil
		}
		b = s.endBlock()
		b.AddEdgeTo(bCond)
		// generate code to test condition
		s.startBlock(bCond)
		if n.Cond != nil {
			s.condBranch(n.Cond, bBody, bEnd, 1)
		} else {
			b := s.endBlock()
			b.Kind = ssa.BlockPlain
			b.AddEdgeTo(bBody)
		}
		s.startBlock(bEnd)
```

Primero inicializamos cada uno de los bloques que componen la sentencia `do-while`: el condicional, el cuerpo, y el bloque que representa el final de la sentencia. Debido a que podemos garantizar que el bucle se ejecuta una vez podemos saltar directamente al cuerpo en lugar de a la condición como en los bucles `for`. Esto lo hacemos agregando un vértice (salto) desde el bloque anterior hacia el cuerpo.  Luego de agregar un poco de lógica para los `break` y `continue` podemos generar el cuerpo de la función. Y por último agregamos un salto desde el cuerpo hacia la condición y luego, dependiendo de si la condición es `nil` nos encargamos de ella, de lo contrario hacemos un ciclo de vuelta hacia el cuerpo de la sentencia.

Ahora finalmente estamos listos para compilar todo el código y usando este comando, podemos generar un archivo HTML donde va a estar el SSA, el AST y otros pasos con acrónimos de 3 letras que Go usa en el medio:

```
GOSSAFUNC=main <carpeta raiz del proyecto>/go/bin/go build  dowhile.go
```

El código que se genera para el programa que hemos venido utilizando es el siguiente. (Go utiliza una ensamblador propio, así que esto no es intel x86 ni nada parecido):

```
MOVL $1, AX
INCQ AX
CMPQ AX, $10
JLT 4
```

¡Bien! Pudimos generar justo el código que queríamos, sin ningun salto directo a la condición, garantizando que el bucle se ejecute al menos una vez.

### Repositorio
El repositorio de este proyecto lo puedes encontrar aquí (github.com/jpelay/go). En la rama `addingdowhile` se encuentra el código de este post, y la rama `addinguntil` contiene una implementación actualizada del post de Eli.

### Conclusiones
El compilador de Go es un proyecto sumamente extenso, que posiblemente lleve mucho tiempo y una gran cantidad de conocimientos sobre compiladores y optimizaciones para poder entender correctamente. Así que mi propósito en este post no fue tratar de entender cada detalle, sino obtener una visión general acerca de como funciona.
Sin duda, la parte más díficil fue tratar de moverme por un proyecto del cual no tengo conocimientos, en un lenguaje que aún no domino y sin nadie a quién pedirle ayuda. Así que el resultado me hace sentir bastante satisfecho, pero sobre todo me impresiona la simpleza de Go y esta simpleza se traslada hacia su compilador, el cuál está muy bien construido, es fácil de seguir y entender y aún así logra ser rápido y eficiente. Un ejemplo de excelente ingienería del software.

