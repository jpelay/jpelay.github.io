## Agregando bucles do-while al compilador de Go

Hace un tiempo me encontré con este [magnífico post](https://eli.thegreenplace.net/2019/go-compiler-internals-adding-a-new-statement-to-go-part-1/) de Eli Benderksy acerca de agregar un nuevo tipo de sentencia a Go. Go es lenguaje que casi no conozco, así que meterme de lleno en el compilador me pareció una excelente idea para conocer más del lenguaje y de cómo funciona un compilador usado en producción. Sin embargo, el post de Eli es viejo y varias de sus pasos se encuentran desfasados, así que este es mi intento de retomar su trabajo y dar los pasos actuales de cómo funciona el compilador.

### Lo qué haremos

Go es un lenguaje minimalista que sólo tiene un tipo de bucle, el cual puede servir de varias formas dependiendo de sus componentes:

{% highlight go %}
for Inic; Cond; Post {
    Cuerpo
}
{% endhighlight %}

Cada uno de los componentes son opcionales, pudiendo hacer bucles while, o infitos usando solo una palabra clave. La simpleza de Go es genial, pero ¿qué tal si queremos hacer un bucle do-while? Una forma de hacerlo usando solo la sintaxis actual del lenguaje es la siguiente:

```go
i := true
for {
    if algunaCondicion {
        i = false
    }
    if !i {
        break
    }
}
```

Esto tiene una desventaja, además de toda la sintaxis extra que estamos lanzando: necesitamos agregar un salto incondicional extra antes de iniciar el bucle, y un salto condicional extra por cada ciclo. Para entender esto, tendríamos que ver el SSA generado por go al momento de compilar. El bucle:

```go
for i := 0; i < 10; {
	i++
}
```

Compila a:

```
        XORL    AX, AX
        JMP     main_pc7
main_pc4:
        INCQ    AX
main_pc7:
        CMPQ    AX, $10
        JLT     main_pc4
```

Esto quiere decir que los pasos que sigue el procesador para ejecutar el bucle son:
1. Ejecuta la inicialización del bucle (`XORL AX, AX`)
2. Salta hacia el condicional (`JMP main_pc7`)
3. Ejecuta la condición, de ser verdad salta hacia el cuerpo del bucle.
4. Ejecuta el cuerpo y el bucle se repite.

Un bucle do-while, por otro lado, puede deshacerse del salto incondicional ya que está garantizado que se puede ejecutar una vez. Nuestra tarea es hacer que el compilador de go parsee y compile correctamente estos bucles.

*(Nota: esto es simplemente un ejercicio, si en serio un salto incondicional es lo suficiente como para afectar el tiempo de ejecución de tu programa, lo más probable es que no quieras usar un lenguaje con recolector de basura como Go).*

### La estructura del compilador
Toda el código del compilador vive en la carpeta `src/cmd/compile/internal`, la cual está dividida en paquetes a los que iré haciendo referenica a lo largo del post. El código de go puede ser clonado fácilmente desde [este mirror en github](https://github.com/golang/go/).

### El analizador léxico

Lo primero que tenemos que hacer es agregar los tokens que el analizador léxico tomára como nuestros bucles do-while. Go tiene una forma más bien extraña de definir sus tokens, estos se encuentran en el archivo syntax/tokens.go el cual contiene un enumerado con los tokens.

```go
_Default // default
_Defer // defer
_Do // do
_While // while
_Else // else
_Fallthrough // fallthrough
```

Para que esta definición sea agregada al compilador debemos instalar la herramienta stringer de go, y usar este comando en la carpeta syntax

```
go generate tokens.go
```

Este comando va a generar el archivo tokens_string.go, que si revisamos podemos notar que contiene dos nuevas entradas en el arreglo de tokens, una para do y otra para while. Ahora podemos tratar de compilar go, para esto vamos a la carpeta src y corremos el comando `./make.bash`. Si tratamos de compilar ahora el compilador se queja:

```
/home/capybara/repos/go/src/hash/crc32/crc32_amd64.go:217:3: syntax error: unexpected do, expecting }

/home/capybara/repos/go/src/hash/crc32/crc32_amd64.go:218:29: syntax error: unexpected do, expecting expression

/home/capybara/repos/go/src/hash/crc32/crc32_amd64.go:219:9: syntax error: unexpected do, expecting expression

/home/capybara/repos/go/src/hash/crc32/crc32_amd64.go:221:2: syntax error: non-declaration statement outside function body

  

go tool dist: FAILED: /home/capybara/repos/go/pkg/tool/linux_amd64/compile -std -pack -o /tmp/go-tool-dist-976817820/hash/crc32/_go_.a -p hash/crc32 -importcfg /tmp/go-tool-dist-976817820/hash/crc32/importcfg -asmhdr /tmp/go-tool-dist-976817820/hash/crc32/go_asm.h -symabis /tmp/go-tool-dist-976817820/hash/crc32/symabis /home/capybara/repos/go/src/hash/crc32/crc32.go /home/capybara/repos/go/src/hash/crc32/crc32_amd64.go /home/capybara/repos/go/src/hash/crc32/crc32_generic.go: exit status 2
```

Ups, parece que en varios sitios dentro del código fuente de go usan la palabra do como nombres para variables. Como en la línea que nos indica el compilador, en el modulo hash:

```go
if len(p) >= 64 {
	left := len(p) & 15
	do := len(p) - left
	crc = ^ieeeCLMUL(^crc, p[:do])
	p = p[do:]
}
```

Para solucionar esto cambiamos el token de `do` a `do_` usamos el comando `go generate` de nuevo e intentamos compilar de nuevo y ¡funciona! Por alguna razón Eli tuvo problemas con el hash de los tokens, pero es problable que haya tenido suerte con los tokens que agregué ya que su hash no choca con el de las otras keywords de go.

### Análisis sintáctico
Luego de convertir el código fuente en una cadena de tokens, el compilador hace el análisis sintáctico. Go usa un análizador sintáctico recursivo descendente bastante estándar. El parser (como también es conocido esta fase), va a leer la cadena de tokens y la convertirá en un árbol de sintaxis concreto (CST), que luego será manipulado en las siguientes fases de compilación y convertido en un AST. Este paso de un CST a un AST quedó de los primeros días del compilador cuando estaba escrito en C y luego fue convertido a Go.

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

Para hacer que go parsee la sentencia debemos ir hacia el archivo syntax/parser.go y agregar un nuevo caso en el `switch` de los tokens:

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

La función en sí es bastante sencilla, primero necesitamos avanzar el token `_Do` para que la función `blockStmt` parsee el cuerpo del bucle. Luego reutilizamos la función `header` que es usada por los bucles `for` y los `if`, y la modificamos un poco para que sea capaz de procesar los bucles do-while. En realidad solo necesitamos agregar un pequeño condicional para que no entrar en los casos extra de sentencias de inicialización y posteriores que sí usan los `for` y los `if`:

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

Para poder imprimir el CST podemos agregar unos print de debug en la función `LoadPackage` del paquete noder. Esta función es la que se encarga de invocar al parser:

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

Y ¡exito! Ya podemos parsear bucles do-while:

```
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

