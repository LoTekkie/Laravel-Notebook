# Useful design patterns in Laravel 

### Table of Contents

- [Repositories](#repositories)
- [API Resources](#api-resources)
- [Factories](#factories)
- [Strategies](#strategies)

___
### Repositories
> A repository can be defined as a layer of abstraction between the domain and data mapping layers, one that provides an avenue of mediation between both, via a collection-like interface for accessing domain objects.

**Rationale:**   
A key benefit of the Repository pattern is that it allows us to use the Principle of Dependency Inversion (or code to abstractions, not concretions). This makes our code more robust to changes, such as if a decision was made later on to switch to a data source that isn't supported by Eloquent. It also helps with keeping the code organized and avoiding duplication, as database-related logic is kept in one place.   

**Sources:**   
* [How to Use the Repository Pattern in a Laravel Application](https://www.twilio.com/blog/repository-pattern-in-laravel-application)
 
**Examples:**

*Repository Contract*

    <?php

    namespace App\Interfaces;

    interface OrderRepositoryInterface 
    {
        public function getAllOrders();
        public function getOrderById($orderId);
        public function deleteOrder($orderId);
        public function createOrder(array $orderDetails);
        public function updateOrder($orderId, array $newDetails);
        public function getFulfilledOrders();
    }

*Respository Implementation*

    <?php

    namespace App\Repositories;

    use App\Interfaces\OrderRepositoryInterface;
    use App\Models\Order;

    class OrderRepository implements OrderRepositoryInterface 
    {
      public function getAllOrders() 
      {
          return Order::all();
      }

      public function getOrderById($orderId) 
      {
          return Order::findOrFail($orderId);
      }

      public function deleteOrder($orderId) 
      {
          Order::destroy($orderId);
      }

      public function createOrder(array $orderDetails) 
      {
          return Order::create($orderDetails);
      }

      public function updateOrder($orderId, array $newDetails) 
      {
          return Order::whereId($orderId)->update($newDetails);
      }

      public function getFulfilledOrders() 
      {
          return Order::where('is_fulfilled', true);
      }
    }

*Repository Execution*

    <?php

    namespace App\Http\Controllers;

    use App\Interfaces\OrderRepositoryInterface;
    use Illuminate\Http\JsonResponse;
    use Illuminate\Http\Request;
    use Illuminate\Http\Response;

    class OrderController extends Controller 
    {
        private OrderRepositoryInterface $orderRepository;

        public function __construct(OrderRepositoryInterface $orderRepository) 
        {
            $this->orderRepository = $orderRepository;
        }

        public function index(): JsonResponse 
        {
            return response()->json([
                'data' => $this->orderRepository->getAllOrders()
            ]);
        }

        public function store(Request $request): JsonResponse 
        {
            $orderDetails = $request->only([
                'client',
                'details'
            ]);

            return response()->json(
                [
                    'data' => $this->orderRepository->createOrder($orderDetails)
                ],
                Response::HTTP_CREATED
            );
        }

        public function show(Request $request): JsonResponse 
        {
            $orderId = $request->route('id');

            return response()->json([
                'data' => $this->orderRepository->getOrderById($orderId)
            ]);
        }

        public function update(Request $request): JsonResponse 
        {
            $orderId = $request->route('id');
            $orderDetails = $request->only([
                'client',
                'details'
            ]);

            return response()->json([
                'data' => $this->orderRepository->updateOrder($orderId, $orderDetails)
            ]);
        }

        public function destroy(Request $request): JsonResponse 
        {
            $orderId = $request->route('id');
            $this->orderRepository->deleteOrder($orderId);

            return response()->json(null, Response::HTTP_NO_CONTENT);
        }
    }

___
### API Resources
> a transformation layer that sits between your Eloquent models and the JSON responses that are actually returned to your application's users.

**Rationale:**  
You may wish to display certain attributes for a subset of users and not others, or you may wish to always include certain relationships in the JSON representation of your models. Eloquent's resource classes allow you to expressively and easily transform your models and model collections into JSON.

**Sources:**   
* [Docs - master - Eloquent: API Resources](https://laravel.com/docs/master/eloquent-resources)

**Examples:**  

*Generating Resources*  

	php artisan make:resource UserResource

*Resource Implementation*

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Resources\Json\JsonResource;

    class UserResource extends JsonResource
    {
        /**
         * Transform the resource into an array.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return array
         */
        public function toArray($request)
        {
            return [
                'id' => $this->id,
                'name' => $this->name,
                'email' => $this->email,
                'created_at' => $this->created_at,
                'updated_at' => $this->updated_at,
            ];
        }
    }

*Resource Execution*  

    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/user/{id}', function ($id) {
        return new UserResource(User::findOrFail($id));
    });


*Generating Resouce Collections*

	php artisan make:resource User --collection
 
	php artisan make:resource UserCollection

*Resource Collection Implementation*  

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Resources\Json\ResourceCollection;

    class UserCollection extends ResourceCollection
    {
        /**
         * Transform the resource collection into an array.
         *
         * @param  \Illuminate\Http\Request  $request
         * @return array
         */
        public function toArray($request)
        {
            return [
                'data' => $this->collection,
                'links' => [
                    'self' => 'link-value',
                ],
            ];
        }
    }


*Resource Collection Execution*  

    use App\Http\Resources\UserCollection;
    use App\Models\User;

    Route::get('/users', function () {
        return new UserCollection(User::all());
    });

___
### Factories
> The Factory pattern is a creational pattern that uses factory methods to deal with the problem of creating objects by specifying their concrete classes. The factory is responsible for creating objects, not the clients. Multiple clients can call the same 
factory.

**Rationale:**  
Factory classes are often implemented because they allow the project to follow the SOLID principles more closely. In particular, the interface segregation and dependency inversion principles.

Factories allow for a lot more long term flexibility. It allows for a more decoupled - and therefore more testable - design.

**Sources:**   
* [A brief overview of design patterns used in laravel - Factory Pattern](https://codesource.io/brief-overview-of-design-pattern-used-in-laravel/#factory-pattern)

**Examples:**  

*Factory Contract*

    interface CarFactoryContract
    {
        public function make(array $carbattery = []): Car;
    }
    
*Factory Implementation*  

    class CarFactory implements CarFactoryContract
    {
        public function make(array $carbattery = [])
        {
            return new Car($carbattery);
        }
    }    

*Factory Execution*  

	$car = (new CarFactory)->make('lithium', 'nickel', 'cobalt');

*Real World (Laravel views)*

    class PostsController
    {
        public function index()
        {
            $posts= Post::all();
            return view('posts.index', ['posts' => $posts]);
        }
    }

    //Illuminate\Support\helpers.php
    /**
     * @return \Illuminate\View\View\Illuminate\Contracts\View\Factory
     * /
    function view($view = null, $data = [], $mergeData = [])
    {
        $factory = app(ViewFactory::class);

        if(func_num_args() === 0){
            return $factory;
        }
        return $factory->make($view, $data, $mergeData);
    }


___
### Strategies
> The strategy pattern is a behavioral pattern. Defines a family of interchangeable algorithms. It always uses an interface. This means the implementation details remain in separate classes. Its programs are to an interface, not an implementation.

**Rationale:**  

**Sources:**   
* []()

**Examples:**  

*Strategy Interface*  

    interface DeliveryStrategy
    {
        public function deliver(Address $address):DeliveryTime;
    }

*Strategy Implentation*  

    class ShipDelivery implements DeliveryStrategy
    {
        public function deliver(Address $addrress):
        {
            $route= new ShipRoute($address);
            echo $route->calculateCosts();
            echo $route->calculateDeliveryTime();
        }
    }

*Strategy Execution*

    class CarDelivery
    {
        public function deliverCar(DeliveryStrategy $strategy, Address $address)
        {
            return $strategy->deliver($address);
        }
    }
    $address = new Address('example address 12345');
    $delivery = new CarDelivery();
    $delivery=>deliver(new ShipDelivery(), $address);

___
### 
> 

**Rationale:**  

**Sources:**   
* []()

**Examples:**  

___
### 
> 

**Rationale:**  

**Sources:**   
* []()

**Examples:**  
