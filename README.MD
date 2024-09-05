# Postgres db sharding with Laravel 10
### references:
1. https://www.citusdata.com/download/
2. https://www.citusdata.com/blog/2023/07/31/schema-based-sharding-comes-to-postgres-with-citus/
## Prerequisites
1. laravel and it's all required dependencies
2. docker installed
3. PgAdmin 4

## Setup
1. make sure your laravel project can access simple postgres service in docker.
    ```bash
    docker compose -f docker-compose.pgsql.yml up -d
    ```
2. run following command to install postgresql with citus
    ```bash
    docker run -d --name citus -p 5432:5432 -e POSTGRES_PASSWORD=password citusdata/citus:12.1
    ```
3. in laravel, set `.env` file for the scheme data
    ```bash
    DB_CONNECTION=pgsql1
    DB_CONNECTION=pgsql2
    DB_HOST=127.0.0.1
    DB_PORT=5432
    DB_DATABASE=postgres
    DB_USERNAME=postgres
    DB_PASSWORD=password
    ```
4. create new migration file in laravel
    ```bash
    php artisan make:migration create_user_table
    ```
5. fill like following example
    ```php
    <?php

    use Illuminate\Database\Migrations\Migration;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    return new class extends Migration
    {
        /**
        * Run the migrations.
        */
        public function up(): void
        {
            Schema::create('users', function (Blueprint $table) {
                $table->id();
                $table->string('name');
                $table->string('email');
            });
        }

        /**
        * Reverse the migrations.
        */
        public function down(): void
        {
            Schema::dropIfExists('users');
        }
    };

    ```
6. Make sure you've already login to postgres in citus. Then create new scheme `tenant_1` and `tenant_2` with pgadmin4
    ![](/scheme-tenant.png)
7. Run migration
    ```bash
    php artisan migrate:fresh --database=pgsql1
    php artisan migrate:fresh --database=pgsql2
    ```
8. Create new model for user that will connected to `tenant_1` and `tenant_2`
    ```bash
    php artisan make:model User1
    php artisan make:model User2
    ```
    ```php
    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Factories\HasFactory;
    use Illuminate\Database\Eloquent\Model;

    class User1 extends Model
    {
        use HasFactory;

        protected $table = 'tenant_1.users';
        protected $primaryKey = 'id';
        public $incrementing = true;
        public $timestamps = false;

        protected $fillable = ['name', 'email'];
    }
    ```
    do the similar way for `tenant_2`
9. below is sample controller for simple create new record to postgres
    ```php
    <?php
    namespace App\Http\Controllers;

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\DB;
    use App\Models\User1;

    class HomeController extends Controller
    {
        public function show() {
            return view('home');
        }
        
        public function create(Request $request) {
            $name = $request->input('name');
            $email = $request->input('email');

            User1::create([
                'name'  => $name,
                'email'  => $email
            ]);

            return to_route('home')->withInput();
        }
    }

    ```