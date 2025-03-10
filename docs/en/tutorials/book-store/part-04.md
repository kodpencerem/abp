# Web Application Development Tutorial - Part 4: Integration Tests
````json
//[doc-params]
{
    "UI": ["MVC","Blazor","BlazorServer","NG"],
    "DB": ["EF","Mongo"]
}
````

````json
//[doc-nav]
{
  "Next": {
    "Name": "Authorization",
    "Path": "tutorials/book-store/part-05"
  },
  "Previous": {
    "Name": "Creating, Updating and Deleting Books",
    "Path": "tutorials/book-store/part-03"
  }
}
````

## About This Tutorial

In this tutorial series, you will build an ABP based web application named `Acme.BookStore`. This application is used to manage a list of books and their authors. It is developed using the following technologies:

* **{{DB_Value}}** as the ORM provider. 
* **{{UI_Value}}** as the UI Framework.

This tutorial is organized as the following parts;

- [Part 1: Creating the server side](part-01.md)
- [Part 2: The book list page](part-02.md)
- [Part 3: Creating, updating and deleting books](part-03.md)
- **Part 4: Integration tests (this part)**
- [Part 5: Authorization](part-05.md)
- [Part 6: Authors: Domain layer](part-06.md)
- [Part 7: Authors: Database Integration](part-07.md)
- [Part 8: Authors: Application Layer](part-08.md)
- [Part 9: Authors: User Interface](part-09.md)
- [Part 10: Book to Author Relation](part-10.md)

### Download the Source Code

This tutorial has multiple versions based on your **UI** and **Database** preferences. We've prepared a few combinations of the source code to be downloaded:

* [MVC (Razor Pages) UI with EF Core](https://github.com/abpframework/abp-samples/tree/master/BookStore-Mvc-EfCore)
* [Blazor UI with EF Core](https://github.com/abpframework/abp-samples/tree/master/BookStore-Blazor-EfCore)
* [Angular UI with MongoDB](https://github.com/abpframework/abp-samples/tree/master/BookStore-Angular-MongoDb)

> If you encounter the "filename too long" or "unzip" error on Windows, please see [this guide](../../kb/windows-path-too-long-fix.md).

{{if UI == "MVC" && DB == "EF"}}

### Video Tutorial

This part is also recorded as a video tutorial and **<a href="https://www.youtube.com/watch?v=aidRB4YFDLM&list=PLsNclT2aHJcPNaCf7Io3DbMN6yAk_DgWJ&index=4" target="_blank">published on YouTube</a>**.

{{end}}

## Test Projects in the Solution

This part covers the **server side** tests. There are several test projects in the solution:

![bookstore-test-projects-v2](./images/bookstore-test-projects-mvc.png)

> Test projects slightly differs based on your UI and Database selection. For example, if you select MongoDB, then the `Acme.BookStore.EntityFrameworkCore.Tests` will be `Acme.BookStore.MongoDB.Tests`.

Each project is used to test the related project. Test projects use the following libraries for testing:

* [Xunit](https://github.com/xunit/xunit) as the main test framework.
* [Shoudly](https://github.com/shouldly/shouldly) as the assertion library.
* [NSubstitute](http://nsubstitute.github.io/) as the mocking library.

{{if DB=="EF"}}

> The test projects are configured to use **SQLite in-memory** as the database. A separate database instance is created and seeded (with the [data seed system](../../framework/infrastructure/data-seeding.md)) to prepare a fresh database for every test.

{{else if DB=="Mongo"}}

> **[EphemeralMongo](https://github.com/asimmon/ephemeral-mongo)** library is used to mock the MongoDB database. A separate database instance is created and seeded (with the [data seed system](../../framework/infrastructure/data-seeding.md)) to prepare a fresh database for every test.

{{end}}

## Adding Test Data

If you had created a data seed contributor as described in the [first part](part-01.md), the same data will be available in your tests. So, you can skip this section. If you haven't created the seed contributor, you can use the `BookStoreTestDataSeedContributor` to seed the same data to be used in the tests below.

## Testing the BookAppService

Add a new test class, named `BookAppService_Tests` in the `Books` namespace (folder) of the `Acme.BookStore.Application.Tests` project:

````csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Shouldly;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Modularity;
using Volo.Abp.Validation;
using Xunit;

namespace Acme.BookStore.Books;

public abstract class BookAppService_Tests<TStartupModule> : BookStoreApplicationTestBase<TStartupModule>
    where TStartupModule : IAbpModule
{
    private readonly IBookAppService _bookAppService;

    protected BookAppService_Tests()
    {
        _bookAppService = GetRequiredService<IBookAppService>();
    }

    [Fact]
    public async Task Should_Get_List_Of_Books()
    {
        //Act
        var result = await _bookAppService.GetListAsync(
            new PagedAndSortedResultRequestDto()
        );

        //Assert
        result.TotalCount.ShouldBeGreaterThan(0);
        result.Items.ShouldContain(b => b.Name == "1984");
    }
}
````

{{if DB == "EF"}}
Add a new implementation class of `BookAppService_Tests` class, named `EfCoreBookAppService_Tests` in the `EntityFrameworkCore\Applications\Books` namespace (folder) of the `Acme.BookStore.EntityFrameworkCore.Tests` project:

````csharp
using Acme.BookStore.Books;
using Xunit;

namespace Acme.BookStore.EntityFrameworkCore.Applications.Books;

[Collection(BookStoreTestConsts.CollectionDefinitionName)]
public class EfCoreBookAppService_Tests : BookAppService_Tests<BookStoreEntityFrameworkCoreTestModule>
{

}
````
{{end}}

{{if DB == "Mongo"}}
Add a new implementation class of `BookAppService_Tests` class, named `MongoDBBookAppService_Tests` in the `MongoDb\Applications\Books` namespace (folder) of the `Acme.BookStore.MongoDB.Tests` project:

````csharp
using Acme.BookStore.MongoDB;
using Acme.BookStore.Books;
using Xunit;

namespace Acme.BookStore.MongoDb.Applications.Books;

[Collection(BookStoreTestConsts.CollectionDefinitionName)]
public class MongoDBBookAppService_Tests : BookAppService_Tests<BookStoreMongoDbTestModule>
{

}
````
{{end}}

* `Should_Get_List_Of_Books` test simply uses `BookAppService.GetListAsync` method to get and check the list of books.
* We can safely check the book "1984" by its name, because we know that this books is available in the database since we've added it in the seed data.

Add a new test method to the `BookAppService_Tests` class that creates a new **valid** book:

````csharp
[Fact]
public async Task Should_Create_A_Valid_Book()
{
    //Act
    var result = await _bookAppService.CreateAsync(
        new CreateUpdateBookDto
        {
            Name = "New test book 42",
            Price = 10,
            PublishDate = DateTime.Now,
            Type = BookType.ScienceFiction
        }
    );

    //Assert
    result.Id.ShouldNotBe(Guid.Empty);
    result.Name.ShouldBe("New test book 42");
}
````

Add a new test that tries to create an invalid book and fails:

````csharp
[Fact]
public async Task Should_Not_Create_A_Book_Without_Name()
{
    var exception = await Assert.ThrowsAsync<AbpValidationException>(async () =>
    {
        await _bookAppService.CreateAsync(
            new CreateUpdateBookDto
            {
                Name = "",
                Price = 10,
                PublishDate = DateTime.Now,
                Type = BookType.ScienceFiction
            }
        );
    });

    exception.ValidationErrors
        .ShouldContain(err => err.MemberNames.Any(mem => mem == "Name"));
}
````

* Since the `Name` is empty, ABP will throw an `AbpValidationException`.

The final test class should be as shown below:

````csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Shouldly;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Modularity;
using Volo.Abp.Validation;
using Xunit;

namespace Acme.BookStore.Books;

public abstract class BookAppService_Tests<TStartupModule> : BookStoreApplicationTestBase<TStartupModule>
    where TStartupModule : IAbpModule
{
    private readonly IBookAppService _bookAppService;

    protected BookAppService_Tests()
    {
        _bookAppService = GetRequiredService<IBookAppService>();
    }

    [Fact]
    public async Task Should_Get_List_Of_Books()
    {
        //Act
        var result = await _bookAppService.GetListAsync(
            new PagedAndSortedResultRequestDto()
        );

        //Assert
        result.TotalCount.ShouldBeGreaterThan(0);
        result.Items.ShouldContain(b => b.Name == "1984");
    }

    [Fact]
    public async Task Should_Create_A_Valid_Book()
    {
        //Act
        var result = await _bookAppService.CreateAsync(
            new CreateUpdateBookDto
            {
                Name = "New test book 42",
                Price = 10,
                PublishDate = DateTime.Now,
                Type = BookType.ScienceFiction
            }
        );

        //Assert
        result.Id.ShouldNotBe(Guid.Empty);
        result.Name.ShouldBe("New test book 42");
    }
    
    [Fact]
    public async Task Should_Not_Create_A_Book_Without_Name()
    {
        var exception = await Assert.ThrowsAsync<AbpValidationException>(async () =>
        {
            await _bookAppService.CreateAsync(
                new CreateUpdateBookDto
                {
                    Name = "",
                    Price = 10,
                    PublishDate = DateTime.Now,
                    Type = BookType.ScienceFiction
                }
            );
        });

        exception.ValidationErrors
            .ShouldContain(err => err.MemberNames.Any(mem => mem == "Name"));
    }
}
````

Open the **Test Explorer Window** (use Test -> Windows -> Test Explorer menu if it is not visible) and **Run All** tests:

![bookstore-appservice-tests](./images/bookstore-appservice-tests.png)

Congratulations, the **green icons** indicates that the tests have been successfully passed!