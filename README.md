# Bookstore API

This is a RESTful API I built for the ElevateLabs Java internship project phase. It's a backend for managing a bookstore's catalog: authors and the books they've written, with full CRUD on both, plus filtering, pagination, and sorting on the books endpoint so it actually behaves like something you'd use in a real catalog rather than a toy CRUD demo.

## What I used

I built this with Java 17 and Spring Boot 3.5.11. Spring Data JPA handles persistence against an H2 in-memory database, so there's zero setup needed to run it, the schema gets created automatically from my entity classes. Validation is handled with Jakarta Bean Validation annotations, and the API is documented with springdoc-openapi, which gives me an interactive Swagger UI for free. I tested everything with Postman as I built it, and I've included the collection in this repo so anyone can import it and try the API themselves.

## How I structured it

I kept this in layers: entities, repositories, a service layer for the actual business logic, and controllers that just handle HTTP concerns and delegate to the services. I think this separation matters even for a small project like this, since it's the structure I'd be expected to explain and extend in a real codebase.

For the Author-Book relationship, I went with a one-directional ManyToOne from Book to Author rather than a bidirectional relationship. A Book knows its Author, but an Author doesn't hold a list of Books. I made this call on purpose: bidirectional JPA relationships are a classic source of infinite recursion when Jackson tries to serialize them to JSON, and dealing with that (via @JsonManagedReference/@JsonBackReference, or separate DTOs in both directions) felt like unnecessary complexity for what this project needs. If I ever need "all books by an author," that's a simple repository query, not something I need baked into the entity itself.

I also didn't let the entities double as request bodies. Creating or updating a book takes a BookRequest with an authorId, not a nested Author object, and the service resolves that id against the AuthorRepository before saving. It's a small thing, but it avoids a whole category of bugs where a client could accidentally create duplicate or orphaned author records just by sending a book payload.

For filtering, I used Spring Data's Specification API rather than a pile of derived query methods. The BookSpecification class builds small, independent predicates, one each for title, genre, author name, and a min/max price range, and the service combines whichever ones are actually present on a given request. Pagination and sorting come from Spring's Pageable, bound automatically from page, size, and sort query parameters. So a single endpoint handles "give me all books," "give me fiction books under 500," and everything in between, without me writing a different method for every combination of filters.

Errors are handled centrally through a GlobalExceptionHandler. A missing author or book returns a clean 404 with a message instead of a stack trace, and validation failures return a 400 with the specific field errors, which is a lot more useful to a client than Spring's default error page.

## Running it

Clone the project, then from the root directory run:

```
mvn spring-boot:run
```

It starts on port 8080. The H2 console is available at `http://localhost:8080/h2-console` if you want to poke around the actual tables (JDBC URL is `jdbc:h2:mem:bookstoredb`, username `sa`, no password). Swagger UI is at `http://localhost:8080/swagger-ui.html` and gives you a full interactive view of every endpoint.

## API overview

Authors live under `/api/authors`. POST creates one, GET returns the full list, GET `/{id}` fetches a single author, PUT `/{id}` updates one, and DELETE `/{id}` removes one.

Books live under `/api/books` and follow the same CRUD shape, but GET also accepts optional query parameters: `title` and `authorName` do a partial, case-insensitive match, `genre` is an exact match, `minPrice` and `maxPrice` bound the price range, and `page`, `size`, and `sort` control pagination and ordering, for example `?genre=Fiction&minPrice=100&sort=price,desc`. All of these are optional and combine freely, so you can filter as little or as much as you want.

## Testing it

Import `postman_collection.json` into Postman and you've got every endpoint ready to go, including a pre-built example of the filtered, paginated, sorted books request so you can see the query parameters in action without writing them by hand. The collection uses a `baseUrl` variable set to `http://localhost:8080`, so it'll work as-is once the app is running locally.

