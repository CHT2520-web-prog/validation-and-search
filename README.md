# User Input Validation and a Search Facility

This practical builds on the previous introduction to Laravel practical to add user input validation and a search facility. 

If you don't have a working Laravel app, complete the [Introduction to Laravel](https://github.com/CHT2520-web-prog/laravel-start/) practical before attempting this practical work. 

## User Input Validation

- Modify the `create()` method in your `FilmController` to call the `validate()` method on the `request` object.

```php
public function store(Request $request)
    {
        $request->validate([
            'title' => ['required'],
            'year' => ['required'],
            'duration' => ['required']
        ]);
        $film = new Film();
        $film->title = $request->title;
        $film->year = $request->year;
        $film->duration = $request->duration;
        $film->save();
        return redirect('/films')->with('success', "'{$film->title}' was added to the database.");
    }
```

- If the validation fails, the _create.blade.php_ will automatically be loaded again so the user can fix the errors. If the validation passes, our controller will continue executing normally.

- Test this in a browser. You should find users can only add a film if they complete all the fields. 

Next, we'll display the validation errors to the user. 

- Open _create.blade.php_ 
- Add code that will display the error messages. 

```html
 <h1>Add a new film</h1>

@if ($errors->any())
    <div class="alert alert-danger">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif

```
- The `$errors` object is automatically created for use by Laravel. All we have to do is loop over the errors t display them for the user.  
- Save your files and check this works in a browser.
- Users should only be able to add a film if they complete all the fields in the form. 

### Test Your Understanding
This simple example demonstrates how user input validation works. However, it could be greatly improved. Try the following: 

1. We have used very simple validation rules i.e. we simply check if the fields have been completed. Modify the rules to make sure
    - The year value is a number greater than 1895 (when cinema was invented).
    - The duration field is a number greater than zero.
    - Use the following to help you: https://laravel.com/framework/docs/validation#available-validation-rules

2. If the form fails validation, the error messages all appear in a group in the page. How can you get the error messages to appear next to the control they relate to. See https://laravel.com/framework/docs/validation#the-at-error-directive for some advice. 

3. If only one field fails validation, it is annoying for the user to re-enter the data for fields that have passed validation. Try and re-populate the form with previous values, see https://laravel.com/framework/docs/validation#repopulating-forms for some advice. 


## A Search Facility
It would make sense to allow users to search the films in the databse, instead of simply displaying them as a list on the homepage. 

- First add a search box to the _layout.blade.php_

```html
<div>
    <form action="/films" method="GET">
        <input type="text" name="search" placeholder="Search Films">
        <button type="submit">Go</button>
    </form>
</div>
```

- Test the page, the search won't work yet, but take note of the url e.g. _films?search=the_ . We are using the same _/films_ route but we have added a search parameter to the querystring. 

- Modify the `index()` method in FilmController so we search films instead of simply grabbing all of them. 

```php
public function index(Request $request)
{
    $search = $request->input('search');
    $films = Film::where('title', 'like', '%' . $search . '%')->get();
    return view('films.index', ['films' => $films]);
}
```

- Test this works. You should now be able to search films by the title. 

Previously, we simply used `Film::all()`, now we are using a 'query builder' see (https://laravel.com/framework/docs/eloquent#building-queries). This allows us to use SQL features such as filtering and ordering.

Previously, when writing plain PHP code, we would have had to construct our own SQL queries and use a prepared statement. 

```sql
$sql= "SELECT * FROM films WHERE title LIKE :search";
$stmt = $pdo->prepare($sql);
$stmt->bindValue(':search', '%' . $search . '%');
$stmt->execute();
$films = $stmt->fetchAll(PDO::FETCH_OBJ);
```

The query builder provides convenient methods (e.g. `where`) so we don't have to work with SQL directly. 


### Pagination
If this was a real web application we might have thousands of films, and a search might return over a hundred films. Displaying all these films in a single page wouldn't be a good experience for the user.

We can split search results over multiple pages using 'pagination'

- In the `index()` method change the query to the following (we simply replace `get` with `paginate`)

```php
$films = Film::where('title', 'like', '%' . $search . '%')->paginate(3)->withQueryString();
```

In _index.blade.php_ ,outside the `for` loop, add the following line that will display the pagination links

```html
 {{ $films->links() }}
 ```

- Test this works. The user should be able to navigate through multiple pages of links. 

The pagination links will look odd because by Laravel assumes we are using the Tailwind CSS framework to style the app.

We can use our own HTML code for the pagination links instead of using the default code.


Create a new file named _pagination.blade.php_ in the _resources/views/components/_ folder.

Add the following code in this file:-

```html
@if ($paginator->hasPages())
    <nav>
        <ul class="pagination">
            {{-- Previous Page Link --}}
            @if ($paginator->onFirstPage())
                <span>@lang('pagination.previous')</span>
            @else
                <a href="{{ $paginator->previousPageUrl() }}" rel="prev">@lang('pagination.previous')</a>
            @endif

            {{-- Next Page Link --}}
            @if ($paginator->hasMorePages())
                <a href="{{ $paginator->nextPageUrl() }}" rel="next">@lang('pagination.next')</a>
            @else
                <span>@lang('pagination.next')</span>
            @endif
        </ul>
    </nav>
@endif
```
This is largely HTML but makes calls to a cursor paginator object to display the correct navigation controls. You can read about the various methods of the cursor pagination class here: https://laravel.com/framework/docs/pagination#cursor-paginator-instance-methods


Next, we need to tell Laravel to use this view file instead of the default one.

- Open _app/Providers/AppServiceProvider.php_.

Import the Paginator class (at the top of the page with the other `use` statements)

```php
use Illuminate\Pagination\Paginator;
```

Modify the `boot()` method to specify our pagination view file

```php
public function boot(): void
    {
        Paginator::defaultView('components.pagination');
    }
```

Test this works, the page should feature simple, unstyled pagination links.

### Test Your Understanding
- A well designed search facility should
    - Give feedback on the number of search results found for the entered search term e.g. _'3 results for "the"'_
    - Let the user know if no search results were found. 
- Modify your code to implement these improvements. 
    - You can use `films->total()` in `index.blade.view` to get the count of films. 

## More Things to Think About
It can be argued that adding validation rules in our controllers violates the 'single responsibility' principle. Generally we should aim to have **skinny** controllers.

A more sophisticated approach to user input validation involves using a custom request object. This allows us to move user input validation rules out of our controller into a dedicated request class. See https://laravel.com/framework/docs/validation#form-request-validation.

Sometimes we need to go beyond the basic validation rules that Laravel provides. For example, we might want to test the year of a film is greater than 1895, but no greater than the current year. To do this we can create custom validation rules. See https://laravel.com/framework/docs/validation#custom-validation-rules









