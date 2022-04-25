## Agregando bucles do-while al compilador de Go

Hace un tiempo me encontré con este [magnífico post](https://eli.thegreenplace.net/2019/go-compiler-internals-adding-a-new-statement-to-go-part-1/) de Eli Benderksy acerca de agregar un nuevo tipo de sentencia a Go. Go es lenguaje que casi no conozco, así que meterme de lleno en el compilador me pareció una excelente idea para conocer más del lenguaje y de cómo funciona un compilador usado en producción. Sin embargo, el post de Eli es viejo y varias de sus pasos se encuentran desfasados, así que este es mi intento de retomar su trabajo y dar los pasos actuales de cómo funciona el compilador.

### Lo qué haremos

Go es un lenguaje minimalista que sólo tiene un tipo de bucle, el cual puede servir de varias formas dependiendo de sus componentes:

```go
for Inic; Cond; Post {
    Cuerpo
}
```
Cada uno de los componentes son opcionales, pudiendo hacer bucles while, o infitos usando solo una palabra clave. La simpleza de Go es genial, pero ¿qué tal si queremos hacer un bucle do-while? Una forma de hacerlo usando solo la sintaxis actual del lenguaje es la siguiente:

```go
i := true
for i {
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

