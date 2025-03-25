

    ```bash
    composer require predis/predis
    # Ou certifique-se que a extensão PHP Redis está instalada e habilitada no servidor
    ```

    * No seu arquivo `.env`:
        ```dotenv
        CACHE_DRIVER=redis
        SESSION_DRIVER=redis # Opcional, mas recomendado para performance

        REDIS_HOST=127.0.0.1
        REDIS_PASSWORD=null
        REDIS_PORT=6379
        REDIS_CLIENT=predis # Ou phpredis se estiver usando a extensão PHP
        ```



```php
<?php

namespace App\Http\Controllers;

use App\Models\Cart; // Modelo hipotético para o carrinho
use App\Models\Product;
use App\Models\Order;
use App\Services\ShippingService; // Serviço hipotético de frete
use App\Services\CouponService;   // Serviço hipotético de cupom
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log; // Para logging
use Illuminate\Support\ItemNotFoundException; // Exemplo de exceção

class CheckoutController extends Controller
{
    protected $shippingService;
    protected $couponService;

    public function __construct(ShippingService $shippingService, CouponService $couponService)
    {
        $this->shippingService = $shippingService;
        $this->couponService = $couponService;
    }

    /**
     * Processa o checkout.
     */
    public function processCheckout(Request $request)
    {
        $user = Auth::user();
        $cartItems = $this->getCartItemsOptimized($user->id); // <- Otimizado aqui

        if ($cartItems->isEmpty()) {
            return redirect()->route('cart.show')->withErrors('Seu carrinho está vazio.');
        }

        // --- Validações (endereço, etc.) ---
        $validatedData = $request->validate([
            'shipping_address_id' => 'required|exists:addresses,id,user_id,' . $user->id,
            'payment_method' => 'required|string',
            'coupon_code' => 'nullable|string',
            // ... outras validações
        ]);

        // --- Cálculos (Subtotal, Frete, Descontos, Total) ---
        // Idealmente, mova essa lógica complexa para um Service ou Action class

        try {
            $calculationData = $this->calculateTotalsOptimized($cartItems, $validatedData, $user); // <- Otimizado aqui

            // --- Processamento de Pagamento (simulado) ---
            // $paymentSuccess = $paymentGateway->charge($calculationData['grand_total'], $validatedData['payment_method'], $user);
            // if (!$paymentSuccess) {
            //     return back()->withErrors('Falha no pagamento.');
            // }

            // --- Criação do Pedido (Transação de Banco de Dados) ---
            $order = DB::transaction(function () use ($user, $cartItems, $calculationData, $validatedData) {

                $order = Order::create([
                    'user_id' => $user->id,
                    'shipping_address_id' => $validatedData['shipping_address_id'],
                    'status' => 'processing', // Ou 'pending_payment' dependendo do fluxo
                    'subtotal' => $calculationData['subtotal'],
                    'shipping_cost' => $calculationData['shipping_cost'],
                    'discount' => $calculationData['discount'],
                    'grand_total' => $calculationData['grand_total'],
                    'payment_method' => $validatedData['payment_method'],
                    'coupon_code' => $calculationData['coupon_code'],
                    // ... outros campos do pedido
                ]);

                // Adiciona itens ao pedido (Query Otimizada - insert em lote se possível/necessário)
                $orderItemsData = $cartItems->map(function ($item) use ($order) {
                    // Busca o preço ATUAL do produto para garantir consistência
                    // Idealmente, o preço já foi validado/cacheado antes
                    $productPrice = $this->getProductPriceOptimized($item->product_id);

                    return [
                        'order_id' => $order->id,
                        'product_id' => $item->product_id,
                        'quantity' => $item->quantity,
                        'price' => $productPrice, // Preço no momento da compra
                        'created_at' => now(),
                        'updated_at' => now(),
                    ];
                })->toArray();

                // Insert em lote é mais eficiente que um loop com create()
                DB::table('order_items')->insert($orderItemsData);

                // Atualizar estoque (pode ser feito via observer ou job)
                // foreach ($cartItems as $item) {
                //     Product::where('id', $item->product_id)->decrement('stock', $item->quantity);
                // }

                // Limpar carrinho do usuário
                Cart::where('user_id', $user->id)->delete();

                // Invalidar caches relevantes (carrinho do usuário)
                Cache::forget("cart_items_user_{$user->id}");
                Cache::forget("cart_totals_user_{$user->id}"); // Exemplo

                return $order;
            });

            // --- Enviar Notificações (Jobs para não bloquear a resposta) ---
            // SendOrderConfirmationEmail::dispatch($order);

            return redirect()->route('order.success', $order->id)->with('status', 'Pedido realizado com sucesso!');

        } catch (\Exception $e) {
            Log::error("Erro no Checkout: " . $e->getMessage(), ['user_id' => $user->id, 'request' => $request->all()]);
            // Reverter pagamento se aplicável (compensação)
            return back()->withErrors('Ocorreu um erro inesperado ao processar seu pedido. Tente novamente mais tarde.');
        }
    }

    // --- Métodos Otimizados ---

    /**
     * Busca itens do carrinho com otimizações.
     * ANTES (Ineficiente - Problema N+1):
     * $cartItems = Cart::where('user_id', $userId)->get();
     * foreach ($cartItems as $item) {
     * // Query separada para cada produto dentro do loop!
     * $productName = $item->product->name;
     * }
     */
    protected function getCartItemsOptimized(int $userId)
    {
        // Chave de cache única para os itens do carrinho deste usuário
        $cacheKey = "cart_items_user_{$userId}";
        $ttl = now()->addMinutes(15); // Tempo de vida do cache (ex: 15 minutos)

        // Tenta buscar do cache, se não encontrar, executa a query e salva no cache
        return Cache::remember($cacheKey, $ttl, function () use ($userId) {
            Log::info("Cache miss para: {$cacheKey}"); // Log para debug
            // Eager Loading: Carrega 'product' junto com 'cart' em poucas queries
            // Select: Seleciona apenas as colunas necessárias
            return Cart::where('user_id', $userId)
                ->select('id', 'user_id', 'product_id', 'quantity') // Seleciona colunas do carrinho
                ->with(['product' => function ($query) {
                    // Seleciona apenas colunas necessárias do produto
                    $query->select('id', 'name', 'price', 'stock', 'image_url', 'weight'); // 'weight' para cálculo de frete
                }])
                ->get();
                // ->filter(function ($item) {
                //      // Filtro extra em PHP se necessário (ex: produto ainda existe)
                //      // Cuidado: Isso carrega tudo e depois filtra na memória
                //      return $item->product !== null;
                // });
        });
    }

    /**
     * Busca o preço atual de um produto (com cache).
     */
    protected function getProductPriceOptimized(int $productId): float
    {
        $cacheKey = "product_price_{$productId}";
        $ttl = now()->addHours(1); // Cache de preço por 1 hora (exemplo)

        return Cache::remember($cacheKey, $ttl, function() use ($productId) {
            Log::info("Cache miss para: {$cacheKey}"); // Log para debug
            $product = Product::select('price')->find($productId);
            if (!$product) {
                 // Lançar exceção ou retornar um valor padrão? Depende da regra de negócio
                 throw new ItemNotFoundException("Produto ID {$productId} não encontrado durante busca de preço.");
            }
            return (float) $product->price;
        });
        // IMPORTANTE: Precisa invalidar esse cache quando o preço do produto mudar! (Veja Observers abaixo)
    }


    /**
     * Calcula totais com otimizações.
     * ANTES: Queries e cálculos poderiam estar espalhados e repetidos.
     */
    protected function calculateTotalsOptimized($cartItems, array $validatedData, $user)
    {
         // Chave de cache pode incluir hash dos itens, cupom, endereço para precisão
         // Mas cuidado com a complexidade da chave e a frequência de mudança
        $cartHash = md5($cartItems->toJson() . $validatedData['shipping_address_id'] . ($validatedData['coupon_code'] ?? ''));
        $cacheKey = "cart_totals_user_{$user->id}_{$cartHash}";
        $ttl = now()->addMinutes(5); // Cache curto para cálculos

        return Cache::remember($cacheKey, $ttl, function() use ($cartItems, $validatedData, $user) {
            Log::info("Cache miss para cálculo de totais: user {$user->id}");

            $subtotal = 0;
            foreach ($cartItems as $item) {
                // Validação de estoque (importante fazer antes de calcular)
                if (!$item->product || $item->product->stock < $item->quantity) {
                    // Invalidar caches e lançar erro para o usuário
                    Cache::forget("cart_items_user_{$user->id}"); // Força re-leitura do carrinho
                    Cache::forget($cacheKey); // Remove este cache de totais
                    throw new \App\Exceptions\OutOfStockException("Produto '{$item->product->name}' fora de estoque ou quantidade insuficiente.");
                }
                 // Usar o preço do produto carregado com eager loading
                $subtotal += $item->quantity * $item->product->price;
            }

            // --- Cálculo de Frete (Exemplo) ---
            // Idealmente, o ShippingService também pode usar cache interno
            $shippingCost = $this->shippingService->calculate(
                $validatedData['shipping_address_id'],
                $cartItems
            );

            // --- Aplicação de Cupom (Exemplo) ---
            $discount = 0;
            $couponCode = $validatedData['coupon_code'] ?? null;
            if ($couponCode) {
                 // CouponService pode usar cache para validar cupons
                $discount = $this->couponService->apply(
                    $couponCode,
                    $user,
                    $subtotal // Passa subtotal para cálculo percentual, etc.
                );
            }

            $grandTotal = $subtotal + $shippingCost - $discount;

            return [
                'subtotal' => $subtotal,
                'shipping_cost' => $shippingCost,
                'discount' => $discount,
                'grand_total' => max(0, $grandTotal), // Evita total negativo
                'coupon_code' => $couponCode, // Retorna o cupom usado (ou null)
            ];
        });
    }
}
```



```php
<?php

namespace App\Observers;

use App\Models\Product;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;

class ProductObserver
{
    /**
     * Handle the Product "saved" event (created or updated).
     *
     * @param  \App\Models\Product  $product
     * @return void
     */
    public function saved(Product $product)
    {
        // Invalida o cache de preço específico deste produto
        $cacheKeyPrice = "product_price_{$product->id}";
        Cache::forget($cacheKeyPrice);
        Log::info("Cache invalidado (saved): {$cacheKeyPrice}");

        // Invalida caches relacionados ao carrinho se o estoque ou preço mudou
        // Isso é mais complexo - pode exigir invalidar todos os caches de carrinho que contêm este produto
        // Uma abordagem mais simples (mas menos granular) seria invalidar caches de totais se algo relevante mudar.
        // if ($product->isDirty('price') || $product->isDirty('stock')) {
        //    // Lógica para invalidar caches de carrinhos/totais afetados
        //    // Ex: Cache::tags(['product:'.$product->id])->flush(); (Requer setup de tags)
        // }
    }

    /**
     * Handle the Product "deleted" event.
     *
     * @param  \App\Models\Product  $product
     * @return void
     */
    public function deleted(Product $product)
    {
        // Invalida o cache de preço
        $cacheKeyPrice = "product_price_{$product->id}";
        Cache::forget($cacheKeyPrice);
        Log::info("Cache invalidado (deleted): {$cacheKeyPrice}");

        // Lógica similar para outros caches relevantes
    }
}
```



```php
<?php

namespace App\Providers;

use App\Models\Product;
use App\Observers\ProductObserver;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    // ... outras propriedades

    /**
     * The model observers for your application.
     *
     * @var array
     */
    protected $observers = [
        Product::class => [ProductObserver::class],
        // Registre observers para outros modelos relevantes (User, Address, Coupon, etc.)
        // Cart::class => [CartObserver::class], // Para invalidar cache de carrinho ao adicionar/remover item
    ];

    // ... boot method
}
```

