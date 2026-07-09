# Camilo Raul Benitez Siguenza 00083223

## Indicaciones

Recientemente, se utilizó AI para crear un sistema de gestion de una biblioteca, el cual ha generado varios errores, su trabajo es arreglarlo. Dado el siguiente caso de uso, explique y/o resuelva cada problema según se le pida.

---

## Consideraciones

La libreria crea automaticamente un correo con los nombres de la persona

---

## Problemas

### 1. Filtro por autor y género (10%)

QA ha reportado que el endpoint para obtener los libros puede filtrar por **autor** y por **género**, o por cualquiera de los dos de manera individual.

Actualmente:

- Filtrar únicamente por autor funciona correctamente.
- Filtrar únicamente por género funciona correctamente.
- Filtrar por **autor y género al mismo tiempo** provoca que el servidor falle.

**Instrucción:** Explique la causa del problema y resuélvalo.

En `BookRepository` el metodo derivado estaba declarado de la siguiente froma:

El atributo `genre` en la entidad `Book` es un enum (`Genre`, mapeado con `@Enumerated(EnumType.STRING)`), pero el metodo del repositorio recibe ese segundo parametro como `String`. Spring Data construye la consulta a partir del nombre del metodo (`author = ?1 AND genre = ?2`) 

A esto se sumaba un segundo error en `BookService.getAllBooks`, donde la llamada invertía el orden de los argumentos: como ambos parámetros eran `String`, esto compilaba sin problema, pero enviaba el género en la posición del autor y viceversa, dando asi un segundo bug.

---

### 2. Error al volver a prestar un libro (10%)

Un usuario reportó que al pedir prestado el libro **The Selfish Gene**, devolverlo e intentar pedirlo prestado nuevamente, el servidor falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

En `MovementService.createMovement`, cuando se registra un préstamo (`BORROWING`):

Pero en la rama de devolucion (`RETURN`), el codigo original solo incrementaba el contador y **nunca volvia a poner `available = true`**:

*The Selfish Gene* tiene `available_count = 1` en `data.sql`. Al prestarlo, el contador baja a 0 y `available` pasa a `false`. Al devolverlo, el contador vuelve a 1, pero `available` se queda en `false` para siempre. En el siguiente intento de préstamo, la validación `if (!book.isAvailable())` lanza `RuntimeException("Book is not available")`, que el `GlobalExceptionHandler` convierte en un `500 Internal Server Error`.

**Solución aplicada**

En la rama de devolución se restablece explícitamente la disponibilidad:

```java
book.setAvailableCount(book.getAvailableCount() + 1);
book.setAvailable(true);
```

---

### 3. Cantidad de libros por género (10%)

Existe un endpoint que devuelve la cantidad de libros disponibles por género. Sin embargo, actualmente dicho endpoint falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

**Causa del problema**

`BookService.getGenresAvailable()` recorre todos los libros y hace:


En `data.sql`, el libro *"The Art of War"* se inserta con `genre = NULL`. El atributo `genre` en la entidad no tiene restricción `nullable = false`, por lo que sí es posible tener un libro sin género en la base de datos. Cuando el bucle llega a ese registro, `book.getGenre()` devuelve `null` y `.name()` lanza un `NullPointerException`, que se traduce en un `500` para todo el endpoint.

**Solución aplicada**

Se omiten del conteo los libros sin género asignado:

```java
for (Book book : books) {
    if (book.getGenre() == null) {
        continue;
    }
    String genreName = book.getGenre().name();
    countByGenre.put(genreName, countByGenre.getOrDefault(genreName, 0L) + 1);
}
```
---

### 4. Error al consultar un libro por ID (10%)

Un miembro del equipo de frontend reporta que la siguiente llamada falla:

```http
GET /books?id=ed16ed1e-7017-4697-a08a-d28c09a74acf
```

**Instrucción:** Explique la causa del problema.


En `BookController` existen dos mapeos distintos bajo `/books`:


El id del libro se expone como **variable de ruta** (`/books/{id}`), no como **query param** (`?id=...`). Al llamar `GET /books?id=...`, la peticion no matchea `/{id}`; matchea el endpoint base `GET /books`, que solo reconoce los query params `author` y `genre`. Como `id` no es un parámetro declarado, Spring simplemente lo ignora (no lanza error), y el método ejecutado es `getAllBooks(null, null)`, que devuelve **la lista completa de libros activos** en un `200 OK`.

```
GET /books/ed16ed1e-7017-4697-a08a-d28c09a74acf
```


### 5. Error al crear un libro (10%)

QA ha reportado que el siguiente payload enviado al endpoint `POST /books` provoca un error:

```json
{
  "title": "Clean Code",
  "author": "Robert C. Martin",
  "genre": "classic",
  "isbn": "978-0132350884",
  "available": true,
  "availableCount": 5
}
```

**Instrucción:** Explique la causa del problema.


El payload envía `"genre": "classic"` (minúsculas). En `BookService.createBook`:

`Genre.valueOf(String)` es **sensible a mayuculas/minusculas** y solo acepta el nombre exacto de una constante del enum (`CLASSIC`, `CRIME`, `SELF_HELP`, etc.). Como `"classic"` no coincide exactamente con `"CLASSIC"`, se lanza `IllegalArgumentException: No enum constant ...Genre.classic`, que el `GlobalExceptionHandler` convierte en `500`.

Es inconsistente con `updateBook`, que sí normaliza el valor antes de convertirlo:

```java
book.setGenre(Genre.valueOf(dto.getGenre().toUpperCase()));
```

`createBook` nunca aplica esa misma normalización, de ahí que falle únicamente al crear.

---

### 6. Devolución de libros no prestados (20%)

QA ha reportado que un usuario es capaz de devolver libros que nunca ha solicitado en préstamo.

**Instrucción:**

- Confirme si este comportamiento es realmente posible.
- Si es posible, explique la causa y resuelva el problema.
- Si no es posible, explique por qué, haciendo referencia al código correspondiente.

Sí, era posible en el código original. `MovementService.createMovement`, para el caso `RETURN`, solo validaba que el lector existiera y que el libro existiera por ISBN:

```java
Lector lector = lectorRepository.findByEmail(dto.getEmail())
        .orElseThrow(() -> new RuntimeException("Lector not found"));

Book book = bookRepository.findByIsbn(dto.getIsbn())
        .orElseThrow(() -> new RuntimeException("Book not found"));
...
} else {
    book.setAvailableCount(book.getAvailableCount() + 1);
}
```
```
No existía ninguna verificación de que el lector tuviera un prestamo (`BORROWING`) activo y sin devolver para ese libro. `MovementRepository` tampoco tenía ningun metodo para consultar el historial de movimientos de un lector/libro (`extends JpaRepository<Movement, UUID> {}` vacio).
Tambien agregu un mteodo en `MovementRepository` para obtener el último movimiento entre un lector y un libro (`findTopByLectorAndBookOrderByTimestampDesc`). Antes de aceptar una devolución, se valida que ese último movimiento exista y sea de tipo `BORROWING`; si no, se lanza un error y no se permite la devolución.


