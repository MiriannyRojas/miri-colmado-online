<script>
  import { onMount } from 'svelte';
	import { createClient } from '@supabase/supabase-js';

const SUPABASE_URL = 'https://iqynujgpuatvxvlzuuwt.supabase.co';

const SUPABASE_PUBLISHABLE_KEY =
  'sb_publishable_4HTuvGBCesz-3iqFqkeTbg_W8ed1JUK';

const supabase = createClient(
  SUPABASE_URL,
  SUPABASE_PUBLISHABLE_KEY
);

  // DATOS DE LA TIENDA
  const STORE_NAME = 'Miri';

  let products = $state([]);
  let categories = $state([]);
  let cart = $state([]);

  let loading = $state(true);
  let error = $state('');
  let message = $state('');

  let search = $state('');
  let selectedCategory = $state('all');

  let user = $state(null);
  let showAuth = $state(false);
  let authMode = $state('login');

  let email = $state('');
  let password = $state('');
  let fullName = $state('');
  let authLoading = $state(false);

  // FORMULARIO DE CONTACTO (TABLA SEPARADA: contact_messages)
  let contactName = $state('');
  let contactEmail = $state('');
  let contactSubject = $state('');
  let contactText = $state('');
  let contactLoading = $state(false);
  let contactStatus = $state('');
  let contactStatusType = $state('success');

  // CARGAR PRODUCTOS Y CATEGORÍAS
  async function loadData() {
    loading = true;
    error = '';

    try {
      // La consulta de productos es independiente de la de categorías,
      // para que un problema con categories no oculte todo el catálogo.
      const productsResult = await supabase
        .from('products')
        .select('*')
        .order('name', { ascending: true });

      if (productsResult.error) throw productsResult.error;

      products = (productsResult.data || []).filter(
        (product) => product.is_available !== false
      );

      const categoriesResult = await supabase
        .from('categories')
        .select('*')
        .order('name', { ascending: true });

      if (categoriesResult.error) {
        console.warn('No se pudieron cargar las categorías:', categoriesResult.error.message);
        categories = [];
      } else {
        categories = categoriesResult.data || [];
      }

      if (products.length === 0) {
        error = 'La conexión funciona, pero la tabla products no tiene productos disponibles o las políticas RLS no permiten leerlos. Revisa Supabase → Table Editor → products y agrega productos; comprueba también las políticas SELECT.';
      }
    } catch (err) {
      console.error('Error cargando productos desde Supabase:', err);
      error = `No se pudieron cargar los productos: ${err?.message || 'error desconocido'}. Verifica que exista la tabla products y que tenga una política SELECT para lectura.`;
    } finally {
      loading = false;
    }
  }

  // INICIAR SESIÓN
  async function handleAuth(event) {
    event.preventDefault();

    authLoading = true;
    message = '';

    try {
      if (authMode === 'register') {
        const { data, error } = await supabase.auth.signUp({
          email,
          password,
          options: {
            data: {
              full_name: fullName
            }
          }
        });

        if (error) throw error;

        if (data.session) {
          user = data.session.user;
          message = '¡Tu cuenta se creó correctamente!';
          showAuth = false;
        } else {
          message =
            'Revisa tu correo para confirmar tu registro antes de iniciar sesión.';
        }
      } else {
        const { data, error } =
          await supabase.auth.signInWithPassword({
            email,
            password
          });

        if (error) throw error;

        user = data.user;
        message = '¡Bienvenido a Miri!';
        showAuth = false;
      }

      password = '';
    } catch (err) {
      message = err.message || 'No fue posible completar la operación.';
    } finally {
      authLoading = false;
    }
  }

  // CERRAR SESIÓN
  async function logout() {
    const { error } = await supabase.auth.signOut();

    if (error) {
      message = error.message;
      return;
    }

    user = null;
    message = 'Has cerrado sesión.';
  }

  // ENVIAR MENSAJE DE CONTACTO A SU PROPIA TABLA
  async function sendContact(event) {
    event.preventDefault();
    contactLoading = true;
    contactStatus = '';

    try {
      const { error } = await supabase.from('contact_messages').insert({
        name: contactName.trim(),
        email: contactEmail.trim(),
        subject: contactSubject.trim(),
        message: contactText.trim()
      });

      if (error) throw error;

      contactStatusType = 'success';
      contactStatus = '¡Gracias! Tu mensaje fue enviado correctamente.';
      contactName = '';
      contactEmail = '';
      contactSubject = '';
      contactText = '';
    } catch (err) {
      console.error('Error guardando el mensaje de contacto:', err);
      contactStatusType = 'error';
      contactStatus = `No se pudo guardar tu mensaje: ${err?.message || 'error desconocido'}. Verifica que exista contact_messages y su política INSERT en Supabase.`;
    } finally {
      contactLoading = false;
    }
  }

  // AGREGAR AL CARRITO
  function addToCart(product) {
    message = '';

    if (Number(product.stock) <= 0) {
      message = 'Este producto no tiene unidades disponibles.';
      return;
    }

    const existing = cart.find((item) => item.id === product.id);

    if (existing) {
      if (existing.quantity >= Number(product.stock)) {
        message = 'No hay más unidades disponibles de este producto.';
        return;
      }

      cart = cart.map((item) =>
        item.id === product.id
          ? { ...item, quantity: item.quantity + 1 }
          : item
      );
    } else {
      cart = [
        ...cart,
        {
          id: product.id,
          name: product.name,
          price: Number(product.price),
          stock: Number(product.stock),
          image_url: product.image_url,
          quantity: 1
        }
      ];
    }

    saveCart();
    message = `${product.name} se agregó al carrito.`;
  }

  // CAMBIAR CANTIDADES
  function changeQuantity(productId, amount) {
    const item = cart.find((product) => product.id === productId);

    if (!item) return;

    const newQuantity = item.quantity + amount;

    if (newQuantity <= 0) {
      cart = cart.filter((product) => product.id !== productId);
    } else if (newQuantity <= item.stock) {
      cart = cart.map((product) =>
        product.id === productId
          ? { ...product, quantity: newQuantity }
          : product
      );
    } else {
      message = 'No hay suficiente inventario.';
      return;
    }

    saveCart();
  }

  function removeFromCart(productId) {
    cart = cart.filter((item) => item.id !== productId);
    saveCart();
  }

  function clearCart() {
    cart = [];
    saveCart();
  }

  // GUARDAR CARRITO EN EL NAVEGADOR
  function saveCart() {
    if (typeof localStorage !== 'undefined') {
      localStorage.setItem('miri-cart', JSON.stringify(cart));
    }
  }

  // FILTRAR PRODUCTOS
  let filteredProducts = $derived.by(() => {
    const searchText = search.toLowerCase().trim();

    return products.filter((product) => {
      const matchesSearch =
        String(product.name || '').toLowerCase().includes(searchText) ||
        String(product.description || '').toLowerCase().includes(searchText);

      const matchesCategory =
        selectedCategory === 'all' ||
        String(product.category_id) === String(selectedCategory);

      return matchesSearch && matchesCategory;
    });
  });

  // TOTALES DEL CARRITO
  let cartCount = $derived(
    cart.reduce((total, item) => total + item.quantity, 0)
  );

  let cartTotal = $derived(
    cart.reduce((total, item) => total + item.price * item.quantity, 0)
  );

  // FORMATEAR PRECIOS DOMINICANOS
  function formatPrice(value) {
    return new Intl.NumberFormat('es-DO', {
      style: 'currency',
      currency: 'DOP'
    }).format(Number(value) || 0);
  }

  // FINALIZAR COMPRA
  function checkoutOrder() {
    if (cart.length === 0) {
      message = 'Agrega productos antes de continuar.';
      return;
    }

    message =
      'Tu carrito está listo, pero el sistema todavía necesita configurar el registro de pedidos online. No se ha enviado ningún pedido.';
  }

  // INICIALIZAR LA PÁGINA
  onMount(() => {
    loadData();

    const savedCart = localStorage.getItem('miri-cart');

    if (savedCart) {
      try {
        cart = JSON.parse(savedCart);
      } catch {
        cart = [];
      }
    }

    supabase.auth.getSession().then(({ data, error }) => {
      if (!error) {
        user = data.session?.user ?? null;
      }
    });

    const {
      data: { subscription }
    } = supabase.auth.onAuthStateChange((_event, session) => {
      user = session?.user ?? null;
    });

    return () => subscription.unsubscribe();
  });
</script>

<svelte:head>
  <title>Miri | Tu colmado de confianza</title>

  <meta
    name="description"
    content="Compra tus productos favoritos en Miri, tu colmado de confianza."
  />

  <meta name="theme-color" content="#176b45" />
</svelte:head>

<header class="header">
  <a class="brand" href="#inicio" aria-label="Inicio de Miri">
    <span class="brand-icon">M</span>

    <span>
      <strong>Miri</strong>
      <small>Tu colmado de confianza</small>
    </span>
  </a>

  <nav>
    <a href="#inicio">Inicio</a>
    <a href="#productos">Productos</a>
    <a href="#nosotros">Nosotros</a>
    <a href="#contacto">Contáctanos</a>
  </nav>

  <div class="header-actions">
    {#if user}
      <span class="user-email">
        {user.email}
      </span>

      <button class="outline-button" onclick={logout}>
        Salir
      </button>
    {:else}
      <button
        class="outline-button"
        onclick={() => {
          showAuth = true;
          authMode = 'login';
          message = '';
        }}
      >
        Mi cuenta
      </button>
    {/if}

    <a class="cart-button" href="#carrito">
      🛒 <span>Carrito ({cartCount})</span>
    </a>
  </div>
</header>

<main>
  <section class="hero" id="inicio">
    <div class="hero-content">
      <span class="eyebrow">TU COLMADO, AHORA ONLINE</span>

      <h1>
        Lo que necesitas,
        <span>más cerca de ti.</span>
      </h1>

      <p>
        Tus productos favoritos en un solo lugar.
        Compra fácil y coordina tu pedido con nosotros.
      </p>

      <a class="primary-button" href="#productos">
        Explorar productos →
      </a>

      <div class="hero-benefits">
        <span>✓ Compra fácil</span>
        <span>✓ Precios en pesos dominicanos</span>
        <span>✓ Prepara tu pedido online</span>
      </div>
    </div>

    <div class="hero-decoration">
      <div class="hero-circle">
        <span class="basket">🛒</span>
        <span class="floating-item item-one">🥛</span>
        <span class="floating-item item-two">🍞</span>
        <span class="floating-item item-three">🍎</span>
      </div>

      <div class="hero-note">
        <span>💚</span>
        <div>
          <strong>¡Bienvenido a Miri!</strong>
          <small>Todo para tu día a día</small>
        </div>
      </div>
    </div>
  </section>

  <section class="benefits">
    <article>
      <span>🛍️</span>
      <div>
        <strong>Compra sencilla</strong>
        <p>Encuentra lo que buscas.</p>
      </div>
    </article>

    <article>
      <span>📱</span>
      <div>
        <strong>Pedido directo</strong>
        <p>Prepara tu pedido desde la web.</p>
      </div>
    </article>

    <article>
      <span>💚</span>
      <div>
        <strong>Cerca de ti</strong>
        <p>La comodidad de tu colmado.</p>
      </div>
    </article>
  </section>

  <section class="products-section" id="productos">
    <div class="section-heading">
      <div>
        <span class="eyebrow">NUESTRO CATÁLOGO</span>
        <h2>Productos para ti</h2>
        <p>Encuentra tus productos favoritos.</p>
      </div>

      <label class="search-box">
        <span>⌕</span>
        <input
          type="search"
          placeholder="Buscar productos..."
          bind:value={search}
          aria-label="Buscar productos"
        />
      </label>
    </div>

    <div class="category-list">
      <button
        class:active={selectedCategory === 'all'}
        onclick={() => (selectedCategory = 'all')}
      >
        Todos
      </button>

      {#each categories as category (category.id)}
        <button
          class:active={selectedCategory === String(category.id)}
          onclick={() => (selectedCategory = String(category.id))}
        >
          {category.name}
        </button>
      {/each}
    </div>

    {#if loading}
      <div class="status-box">
        <span class="loader"></span>
        <p>Cargando productos desde Supabase...</p>
      </div>
    {:else if error}
      <div class="status-box error-box">
        <span>⚠️</span>
        <p>{error}</p>
        <button class="primary-button" onclick={loadData}>
          Intentar de nuevo
        </button>
      </div>
    {:else if filteredProducts.length === 0}
      <div class="status-box">
        <span>🛍️</span>
        <h3>No encontramos productos</h3>
        <p>
          Prueba otra búsqueda o agrega productos en tu base de datos.
        </p>
      </div>
    {:else}
      <div class="product-grid">
        {#each filteredProducts as product (product.id)}
          <article class="product-card">
            <div class="product-image">
              {#if product.image_url}
                <img
                  src={product.image_url}
                  alt={product.name}
                  loading="lazy"
                />
              {:else}
                <span>🛍️</span>
              {/if}

              {#if Number(product.stock) > 0}
                <span class="stock-label">Disponible</span>
              {:else}
                <span class="stock-label unavailable">Agotado</span>
              {/if}
            </div>

            <div class="product-info">
              <span class="product-category">
                {categories.find(
                  (category) =>
                    String(category.id) === String(product.category_id)
                )?.name || 'Colmado'}
              </span>

              <h3>{product.name}</h3>

              <p class="product-description">
                {product.description || 'Producto disponible en Miri.'}
              </p>

              <div class="product-bottom">
                <div>
                  <strong class="price">
                    {formatPrice(product.price)}
                  </strong>

                  <small>
                    {Number(product.stock) > 0
                      ? `${product.stock} disponibles`
                      : 'Sin existencias'}
                  </small>
                </div>

                <button
                  class="add-button"
                  disabled={Number(product.stock) <= 0}
                  onclick={() => addToCart(product)}
                  aria-label={`Agregar ${product.name} al carrito`}
                >
                  +
                </button>
              </div>
            </div>
          </article>
        {/each}
      </div>
    {/if}
  </section>

  <section class="cart-section" id="carrito">
    <div class="cart-heading">
      <div>
        <span class="eyebrow">TU SELECCIÓN</span>
        <h2>Mi carrito 🛒</h2>
      </div>

      {#if cart.length > 0}
        <button class="text-button" onclick={clearCart}>
          Vaciar carrito
        </button>
      {/if}
    </div>

    {#if cart.length === 0}
      <div class="empty-cart">
        <span>🛍️</span>
        <h3>Tu carrito está vacío</h3>
        <p>Agrega productos del catálogo para preparar tu pedido.</p>
        <a href="#productos" class="primary-button">
          Ver productos
        </a>
      </div>
    {:else}
      <div class="cart-content">
        <div class="cart-items">
          {#each cart as item (item.id)}
            <article class="cart-item">
              <div class="cart-item-image">
                {#if item.image_url}
                  <img src={item.image_url} alt={item.name} />
                {:else}
                  🛍️
                {/if}
              </div>

              <div class="cart-item-info">
                <strong>{item.name}</strong>
                <span>{formatPrice(item.price)} por unidad</span>

                <div class="quantity-control">
                  <button
                    aria-label="Disminuir cantidad"
                    onclick={() => changeQuantity(item.id, -1)}
                  >
                    −
                  </button>

                  <span>{item.quantity}</span>

                  <button
                    aria-label="Aumentar cantidad"
                    onclick={() => changeQuantity(item.id, 1)}
                  >
                    +
                  </button>
                </div>
              </div>

              <div class="cart-item-end">
                <strong>
                  {formatPrice(item.price * item.quantity)}
                </strong>

                <button
                  class="remove-button"
                  onclick={() => removeFromCart(item.id)}
                >
                  Eliminar
                </button>
              </div>
            </article>
          {/each}
        </div>

        <aside class="cart-summary">
          <h3>Resumen del pedido</h3>

          <div class="summary-line">
            <span>Productos</span>
            <span>{cartCount} unidades</span>
          </div>

          <div class="summary-line total-line">
            <strong>Total estimado</strong>
            <strong>{formatPrice(cartTotal)}</strong>
          </div>

          <p>
            El total es estimado. El colmado confirmará la disponibilidad,
            el precio y cualquier costo de entrega.
          </p>

          <button class="whatsapp-button" onclick={checkoutOrder}>
            Finalizar compra
          </button>
        </aside>
      </div>
    {/if}
  </section>

  <section class="about-section" id="nosotros">
    <div class="about-icon">💚</div>

    <div>
      <span class="eyebrow">SOMOS MIRI</span>
      <h2>Tu colmado de confianza.</h2>

      <p>
        Queremos hacer tus compras más sencillas, con un catálogo práctico
        para que puedas consultar productos y preparar tus pedidos desde
        donde estés.
      </p>
    </div>
  </section>

  <section class="contact-section" id="contacto">
    <div class="contact-intro">
      <span class="eyebrow">ESTAMOS PARA AYUDARTE</span>
      <h2>Contáctanos</h2>
      <p>
        ¿Tienes alguna pregunta, sugerencia o consulta sobre un producto?
        Envíanos un mensaje y nuestro equipo podrá revisarlo.
      </p>
      <div class="contact-note">
        <span>✉️</span>
        <div>
          <strong>Tu mensaje importa</strong>
          <p>Los mensajes se guardan de forma separada de las cuentas de usuario.</p>
        </div>
      </div>
    </div>

    <form class="contact-form" onsubmit={sendContact}>
      <label for="contactName">Nombre completo</label>
      <input
        id="contactName"
        type="text"
        bind:value={contactName}
        placeholder="Escribe tu nombre"
        required
        minlength="2"
        autocomplete="name"
      />

      <label for="contactEmail">Correo electrónico</label>
      <input
        id="contactEmail"
        type="email"
        bind:value={contactEmail}
        placeholder="tucorreo@ejemplo.com"
        required
        autocomplete="email"
      />

      <label for="contactSubject">Asunto</label>
      <input
        id="contactSubject"
        type="text"
        bind:value={contactSubject}
        placeholder="¿Sobre qué nos escribes?"
        required
        minlength="3"
      />

      <label for="contactText">Mensaje</label>
      <textarea
        id="contactText"
        bind:value={contactText}
        placeholder="Escribe aquí tu mensaje..."
        required
        minlength="10"
        rows="5"
      ></textarea>

      <button class="primary-button contact-submit" type="submit" disabled={contactLoading}>
        {contactLoading ? 'Enviando...' : 'Enviar mensaje →'}
      </button>

      {#if contactStatus}
        <p class:contact-error={contactStatusType === 'error'} class:contact-success={contactStatusType === 'success'} role="status">
          {contactStatus}
        </p>
      {/if}
    </form>
  </section>
</main>

<footer>
  <a class="footer-brand" href="#inicio">Miri<span>.</span></a>

  <p>Tu colmado de confianza.</p>

  <span>© {new Date().getFullYear()} Miri. Todos los derechos reservados.</span>
</footer>

{#if showAuth}
  <div
    class="modal-backdrop"
    role="presentation"
    onclick={(event) => {
      if (event.target === event.currentTarget) showAuth = false;
    }}
  >
    <section
      class="auth-modal"
      role="dialog"
      aria-modal="true"
      aria-labelledby="auth-title"
    >
      <button
        class="close-modal"
        aria-label="Cerrar"
        onclick={() => (showAuth = false)}
      >
        ×
      </button>

      <div class="auth-logo">M</div>

      <h2 id="auth-title">
        {authMode === 'login' ? '¡Qué bueno verte!' : 'Únete a Miri'}
      </h2>

      <p>
        {authMode === 'login'
          ? 'Inicia sesión en tu cuenta.'
          : 'Crea tu cuenta para comenzar.'}
      </p>

      <form onsubmit={handleAuth}>
        {#if authMode === 'register'}
          <label for="fullName">Nombre completo</label>

          <input
            id="fullName"
            type="text"
            bind:value={fullName}
            placeholder="Tu nombre"
            required
            minlength="2"
            autocomplete="name"
          />
        {/if}

        <label for="email">Correo electrónico</label>

        <input
          id="email"
          type="email"
          bind:value={email}
          placeholder="tucorreo@ejemplo.com"
          required
          autocomplete="email"
        />

        <label for="password">Contraseña</label>

        <input
          id="password"
          type="password"
          bind:value={password}
          placeholder="Mínimo 6 caracteres"
          required
          minlength="6"
          autocomplete={
            authMode === 'login' ? 'current-password' : 'new-password'
          }
        />

        <button class="primary-button auth-submit" disabled={authLoading}>
          {authLoading
            ? 'Procesando...'
            : authMode === 'login'
              ? 'Iniciar sesión'
              : 'Crear cuenta'}
        </button>
      </form>

      <p class="auth-switch">
        {authMode === 'login'
          ? '¿Todavía no tienes cuenta?'
          : '¿Ya tienes una cuenta?'}

        <button
          onclick={() => {
            authMode = authMode === 'login' ? 'register' : 'login';
            message = '';
          }}
        >
          {authMode === 'login' ? 'Regístrate' : 'Inicia sesión'}
        </button>
      </p>
    </section>
  </div>
{/if}

{#if message}
  <div class="toast" role="status">
    <span>{message}</span>

    <button
      aria-label="Cerrar mensaje"
      onclick={() => (message = '')}
    >
      ×
    </button>
  </div>
{/if}

<style>
  :global(*) {
    box-sizing: border-box;
  }

  :global(html) {
    scroll-behavior: smooth;
    scroll-padding-top: 90px;
  }

  :global(body) {
    margin: 0;
    font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont,
      "Segoe UI", sans-serif;
    color: #23352b;
    background: #fffdf8;
  }

  :global(button),
  :global(input),
  :global(textarea) {
    font: inherit;
  }

  :global(button),
  :global(a) {
    -webkit-tap-highlight-color: transparent;
  }

  :global(a) {
    color: inherit;
    text-decoration: none;
  }

  button {
    cursor: pointer;
  }

  button:disabled {
    cursor: not-allowed;
    opacity: 0.5;
  }

  .header {
    min-height: 82px;
    padding: 15px 6%;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 25px;
    background: #fffdf8;
    border-bottom: 1px solid #e9eee7;
    position: sticky;
    top: 0;
    z-index: 20;
  }

  .brand {
    display: flex;
    align-items: center;
    gap: 11px;
  }

  .brand-icon,
  .auth-logo {
    display: grid;
    place-items: center;
    width: 45px;
    height: 45px;
    border-radius: 15px;
    color: white;
    background: #176b45;
    font-size: 25px;
    font-weight: 850;
  }

  .brand strong {
    display: block;
    font-size: 23px;
    letter-spacing: -1px;
  }

  .brand small {
    display: block;
    margin-top: 2px;
    color: #7a887e;
    font-size: 11px;
  }

  nav {
    display: flex;
    gap: 30px;
    color: #58675d;
    font-size: 14px;
  }

  nav a:hover {
    color: #176b45;
  }

  .header-actions {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .outline-button,
  .cart-button,
  .primary-button,
  .whatsapp-button {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    border-radius: 12px;
    padding: 12px 17px;
    font-size: 13px;
    font-weight: 750;
    border: 1px solid transparent;
    transition: 0.2s ease;
  }

  .outline-button {
    background: transparent;
    border-color: #dbe5dc;
    color: #176b45;
  }

  .outline-button:hover {
    background: #eff6ef;
  }

  .cart-button,
  .primary-button {
    color: white;
    background: #176b45;
  }

  .cart-button:hover,
  .primary-button:hover {
    background: #105337;
    transform: translateY(-1px);
  }

  .user-email {
    max-width: 130px;
    overflow: hidden;
    text-overflow: ellipsis;
    color: #64746a;
    font-size: 12px;
  }

  main {
    width: min(1180px, 90%);
    margin: 0 auto;
  }

  .hero {
    display: grid;
    grid-template-columns: 1.05fr 0.95fr;
    align-items: center;
    gap: 50px;
    min-height: 490px;
    padding: 55px 0 60px;
  }

  .eyebrow {
    display: inline-block;
    color: #278254;
    font-size: 11px;
    font-weight: 850;
    letter-spacing: 2px;
  }

  .hero h1 {
    margin: 18px 0;
    max-width: 600px;
    color: #1b3527;
    font-size: clamp(40px, 5vw, 65px);
    line-height: 1.06;
    letter-spacing: -3px;
  }

  .hero h1 span {
    display: block;
    color: #238354;
  }

  .hero-content > p {
    max-width: 490px;
    color: #748176;
    line-height: 1.8;
    font-size: 15px;
  }

  .hero-content > .primary-button {
    margin-top: 15px;
    padding: 15px 22px;
  }

  .hero-benefits {
    display: flex;
    flex-wrap: wrap;
    gap: 13px;
    margin-top: 28px;
    color: #64756a;
    font-size: 11px;
  }

  .hero-decoration {
    position: relative;
    display: grid;
    place-items: center;
    min-height: 350px;
    border-radius: 45% 40% 35% 45%;
    background: #eaf3e5;
  }

  .hero-circle {
    display: grid;
    place-items: center;
    position: relative;
    width: 270px;
    height: 270px;
    border-radius: 50%;
    background: #d8e8d0;
  }

  .basket {
    font-size: 125px;
    filter: drop-shadow(0 15px 12px #8a9f8833);
  }

  .floating-item {
    display: grid;
    place-items: center;
    position: absolute;
    width: 62px;
    height: 62px;
    border-radius: 20px;
    background: white;
    font-size: 30px;
    box-shadow: 0 10px 30px #27472d16;
  }

  .item-one {
    top: 10px;
    left: 4px;
    transform: rotate(-10deg);
  }

  .item-two {
    top: 45px;
    right: -14px;
    transform: rotate(9deg);
  }

  .item-three {
    bottom: 5px;
    left: 25px;
    transform: rotate(9deg);
  }

  .hero-note {
    display: flex;
    align-items: center;
    gap: 12px;
    position: absolute;
    bottom: 22px;
    left: 12px;
    padding: 15px 19px;
    border-radius: 16px;
    background: white;
    box-shadow: 0 12px 35px #203e2814;
  }

  .hero-note > span {
    font-size: 27px;
  }

  .hero-note strong,
  .hero-note small {
    display: block;
  }

  .hero-note strong {
    font-size: 13px;
  }

  .hero-note small {
    margin-top: 5px;
    color: #849087;
    font-size: 11px;
  }

  .benefits {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 18px;
    padding: 25px;
    border: 1px solid #e7eee4;
    border-radius: 22px;
    background: white;
  }

  .benefits article {
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 8px;
  }

  .benefits article > span {
    display: grid;
    place-items: center;
    width: 48px;
    height: 48px;
    border-radius: 15px;
    background: #eff6e9;
    font-size: 22px;
  }

  .benefits strong {
    font-size: 13px;
  }

  .benefits p {
    margin: 5px 0 0;
    color: #849087;
    font-size: 11px;
  }

  .products-section,
  .cart-section {
    padding: 85px 0 25px;
  }

  .section-heading,
  .cart-heading {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 25px;
  }

  .section-heading h2,
  .cart-heading h2,
  .about-section h2 {
    margin: 9px 0;
    color: #1b3527;
    font-size: clamp(28px, 3vw, 38px);
    letter-spacing: -1.2px;
  }

  .section-heading p {
    margin: 0;
    color: #839087;
    font-size: 13px;
  }

  .search-box {
    display: flex;
    align-items: center;
    gap: 10px;
    width: min(320px, 100%);
    padding: 13px 15px;
    border: 1px solid #e2e9df;
    border-radius: 13px;
    background: white;
  }

  .search-box span {
    font-size: 25px;
    color: #6b8b73;
  }

  .search-box input {
    width: 100%;
    border: 0;
    outline: 0;
    background: transparent;
    color: #23352b;
    font-size: 13px;
  }

  .category-list {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin: 30px 0;
  }

  .category-list button {
    padding: 10px 17px;
    border: 1px solid #e3eae0;
    border-radius: 30px;
    background: white;
    color: #6c7b70;
    font-size: 12px;
    font-weight: 650;
  }

  .category-list button.active {
    border-color: #176b45;
    background: #176b45;
    color: white;
  }

  .product-grid {
    display: grid;
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 20px;
  }

  .product-card {
    overflow: hidden;
    border: 1px solid #e9eee6;
    border-radius: 18px;
    background: white;
    transition: transform 0.2s, box-shadow 0.2s;
  }

  .product-card:hover {
    transform: translateY(-4px);
    box-shadow: 0 14px 35px #294a2c0e;
  }

  .product-image {
    position: relative;
    display: grid;
    place-items: center;
    height: 185px;
    overflow: hidden;
    background: #f1f5eb;
  }

  .product-image > span:first-child {
    font-size: 60px;
  }

  .product-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .stock-label {
    position: absolute;
    top: 12px;
    left: 12px;
    padding: 6px 9px;
    border-radius: 20px;
    background: #e5f5e5;
    color: #287444;
    font-size: 10px;
    font-weight: 800;
  }

  .stock-label.unavailable {
    background: #fae6e3;
    color: #a63e34;
  }

  .product-info {
    padding: 17px;
  }

  .product-category {
    color: #2b8654;
    font-size: 10px;
    font-weight: 800;
    text-transform: uppercase;
    letter-spacing: 1px;
  }

  .product-info h3 {
    margin: 8px 0;
    color: #24382b;
    font-size: 15px;
  }

  .product-description {
    min-height: 35px;
    margin: 0;
    color: #89948b;
    font-size: 11px;
    line-height: 1.6;
  }

  .product-bottom {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    margin-top: 18px;
  }

  .product-bottom strong,
  .product-bottom small {
    display: block;
  }

  .price {
    color: #176b45;
    font-size: 16px;
  }

  .product-bottom small {
    margin-top: 5px;
    color: #94a096;
    font-size: 10px;
  }

  .add-button {
    display: grid;
    place-items: center;
    width: 39px;
    height: 39px;
    border: 0;
    border-radius: 12px;
    background: #176b45;
    color: white;
    font-size: 25px;
  }

  .add-button:hover {
    background: #105337;
  }

  .status-box,
  .empty-cart {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 220px;
    padding: 35px;
    border: 1px dashed #dce7d9;
    border-radius: 20px;
    background: white;
    text-align: center;
  }

  .status-box > span,
  .empty-cart > span {
    font-size: 40px;
  }

  .status-box p,
  .empty-cart p {
    color: #819087;
    font-size: 13px;
    line-height: 1.7;
  }

  .error-box {
    color: #8d302b;
  }

  .loader {
    width: 32px;
    height: 32px;
    border: 3px solid #dce9db;
    border-top-color: #176b45;
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .cart-section {
    padding-bottom: 80px;
  }

  .cart-heading {
    margin-bottom: 25px;
  }

  .text-button {
    border: 0;
    background: transparent;
    color: #b94b42;
    font-size: 12px;
    font-weight: 750;
  }

  .empty-cart h3 {
    margin-bottom: 0;
  }

  .empty-cart .primary-button {
    margin-top: 10px;
  }

  .cart-content {
    display: grid;
    grid-template-columns: 1.5fr 0.8fr;
    align-items: start;
    gap: 25px;
  }

  .cart-items {
    display: grid;
    gap: 12px;
  }

  .cart-item {
    display: flex;
    align-items: center;
    gap: 15px;
    padding: 17px;
    border: 1px solid #e6ede3;
    border-radius: 17px;
    background: white;
  }

  .cart-item-image {
    display: grid;
    place-items: center;
    flex-shrink: 0;
    width: 72px;
    height: 72px;
    overflow: hidden;
    border-radius: 13px;
    background: #f0f5eb;
    font-size: 25px;
  }

  .cart-item-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .cart-item-info {
    flex: 1;
    min-width: 0;
  }

  .cart-item-info strong,
  .cart-item-info > span {
    display: block;
  }

  .cart-item-info strong {
    font-size: 13px;
  }

  .cart-item-info > span {
    margin-top: 5px;
    color: #829087;
    font-size: 11px;
  }

  .quantity-control {
    display: inline-flex;
    align-items: center;
    gap: 14px;
    margin-top: 10px;
    padding: 4px;
    border: 1px solid #e4ebe0;
    border-radius: 9px;
  }

  .quantity-control button {
    width: 25px;
    height: 25px;
    border: 0;
    border-radius: 6px;
    background: #eff5ec;
    color: #176b45;
  }

  .quantity-control span {
    font-size: 12px;
  }

  .cart-item-end {
    display: grid;
    justify-items: end;
    gap: 12px;
  }

  .cart-item-end strong {
    font-size: 13px;
  }

  .remove-button {
    border: 0;
    background: transparent;
    color: #b94b42;
    font-size: 11px;
  }

  .cart-summary {
    padding: 23px;
    border: 1px solid #e2eade;
    border-radius: 19px;
    background: white;
  }

  .cart-summary h3 {
    margin-top: 0;
    font-size: 18px;
  }

  .summary-line {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    padding: 16px 0;
    border-bottom: 1px solid #edf0eb;
    color: #78857b;
    font-size: 12px;
  }

  .total-line {
    color: #1b3527;
    font-size: 14px;
  }

  .total-line strong:last-child {
    color: #176b45;
    font-size: 18px;
  }

  .cart-summary > p {
    color: #879188;
    font-size: 11px;
    line-height: 1.7;
  }

  .whatsapp-button {
    width: 100%;
    margin-top: 12px;
    border: 0;
    background: #218c53;
    color: white;
  }

  .whatsapp-button:hover {
    background: #176d40;
  }

  .about-section {
    display: flex;
    align-items: center;
    gap: 30px;
    margin-bottom: 75px;
    padding: 45px;
    border-radius: 25px;
    background: #eaf3e5;
  }

  .about-icon {
    display: grid;
    place-items: center;
    flex-shrink: 0;
    width: 95px;
    height: 95px;
    border-radius: 25px;
    background: white;
    font-size: 42px;
  }

  .about-section h2 {
    font-size: 30px;
  }

  .about-section p {
    max-width: 650px;
    color: #6d7d70;
    font-size: 13px;
    line-height: 1.9;
  }

  .contact-section {
    display: grid;
    grid-template-columns: 0.9fr 1.1fr;
    gap: 55px;
    align-items: start;
    margin: 55px 0 85px;
    padding: 42px;
    border: 1px solid #e5ede2;
    border-radius: 25px;
    background: #f5f8f1;
  }

  .contact-intro h2 {
    margin: 10px 0 15px;
    color: #1b3527;
    font-size: clamp(30px, 3vw, 40px);
    letter-spacing: -1.2px;
  }

  .contact-intro > p {
    color: #748176;
    font-size: 14px;
    line-height: 1.8;
  }

  .contact-note {
    display: flex;
    align-items: flex-start;
    gap: 13px;
    margin-top: 28px;
    padding: 17px;
    border-radius: 15px;
    background: white;
  }

  .contact-note > span {
    font-size: 24px;
  }

  .contact-note strong {
    font-size: 13px;
  }

  .contact-note p {
    margin: 5px 0 0;
    color: #748176;
    font-size: 12px;
    line-height: 1.6;
  }

  .contact-form {
    display: grid;
    gap: 10px;
    padding: 25px;
    border: 1px solid #e5ede2;
    border-radius: 18px;
    background: white;
  }

  .contact-form label {
    margin-top: 5px;
    color: #34483a;
    font-size: 12px;
    font-weight: 750;
  }

  .contact-form input,
  .contact-form textarea {
    width: 100%;
    padding: 13px 14px;
    border: 1px solid #dfe7dc;
    border-radius: 10px;
    outline: none;
    background: #fff;
    color: #23352b;
    font-size: 13px;
    resize: vertical;
  }

  .contact-form input:focus,
  .contact-form textarea:focus {
    border-color: #278254;
    box-shadow: 0 0 0 3px #27825418;
  }

  .contact-submit {
    margin-top: 8px;
    border: 0;
  }

  .contact-success,
  .contact-error {
    margin: 8px 0 0;
    padding: 12px;
    border-radius: 10px;
    font-size: 12px;
    line-height: 1.6;
    overflow-wrap: anywhere;
  }

  .contact-success {
    background: #edf8ed;
    color: #176b45;
  }

  .contact-error {
    background: #fff0ed;
    color: #a52d20;
  }

  footer {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
    padding: 30px 6%;
    border-top: 1px solid #e8eee5;
    color: #859087;
    font-size: 11px;
  }

  .footer-brand {
    color: #176b45;
    font-size: 25px;
    font-weight: 850;
  }

  .footer-brand span {
    color: #b9d68b;
  }

  .modal-backdrop {
    position: fixed;
    inset: 0;
    z-index: 100;
    display: grid;
    place-items: center;
    overflow-y: auto;
    padding: 20px;
    background: #132b1dcc;
  }

  .auth-modal {
    position: relative;
    width: min(420px, 100%);
    padding: 35px;
    border-radius: 24px;
    background: #fffdf8;
    box-shadow: 0 20px 70px #0002;
  }

  .close-modal {
    position: absolute;
    top: 15px;
    right: 17px;
    width: 35px;
    height: 35px;
    border: 0;
    border-radius: 10px;
    background: #eff4eb;
    color: #3a6045;
    font-size: 25px;
  }

  .auth-logo {
    width: 48px;
    height: 48px;
    margin-bottom: 20px;
  }

  .auth-modal h2 {
    margin: 0;
    color: #1b3527;
    font-size: 27px;
    letter-spacing: -1px;
  }

  .auth-modal > p {
    color: #7d8a80;
    font-size: 12px;
    line-height: 1.7;
  }

  .auth-modal form {
    display: grid;
    gap: 10px;
    margin-top: 25px;
  }

  .auth-modal label {
    margin-top: 8px;
    color: #536457;
    font-size: 12px;
    font-weight: 750;
  }

  .auth-modal input {
    width: 100%;
    padding: 13px;
    border: 1px solid #dfe8dc;
    border-radius: 10px;
    outline-color: #3c8d5d;
    background: white;
  }

  .auth-submit {
    width: 100%;
    margin-top: 12px;
  }

  .auth-switch {
    margin-top: 22px;
    text-align: center;
    font-size: 12px;
  }

  .auth-switch button {
    border: 0;
    background: transparent;
    color: #176b45;
    font-weight: 850;
  }

  .toast {
    position: fixed;
    right: 20px;
    bottom: 20px;
    z-index: 120;
    display: flex;
    align-items: center;
    gap: 15px;
    max-width: min(420px, calc(100vw - 40px));
    padding: 15px 18px;
    border: 1px solid #dbe9d7;
    border-radius: 14px;
    background: white;
    box-shadow: 0 12px 35px #233d2518;
    color: #315b3c;
    font-size: 12px;
    line-height: 1.6;
  }

  .toast button {
    border: 0;
    background: transparent;
    color: #52745a;
    font-size: 20px;
  }

  @media (max-width: 950px) {
    .header {
      flex-wrap: wrap;
    }

    .header nav {
      order: 3;
      width: 100%;
      justify-content: center;
      padding-top: 10px;
    }

    .hero {
      gap: 25px;
    }

    .hero-decoration {
      min-height: 300px;
    }

    .hero-circle {
      width: 220px;
      height: 220px;
    }

    .basket {
      font-size: 100px;
    }

    .product-grid {
      grid-template-columns: repeat(3, minmax(0, 1fr));
    }

    .user-email {
      display: none;
    }
  }

  @media (max-width: 650px) {
    .header {
      padding: 12px 5%;
      gap: 12px;
    }

    .brand-icon {
      width: 39px;
      height: 39px;
    }

    .brand strong {
      font-size: 20px;
    }

    .brand small {
      font-size: 9px;
    }

    .header-actions {
      gap: 6px;
    }

    .outline-button,
    .cart-button {
      padding: 10px;
      font-size: 11px;
    }

    .cart-button span {
      display: none;
    }

    nav {
      gap: 25px;
      font-size: 12px;
    }

    main {
      width: 90%;
    }

    .hero {
      grid-template-columns: 1fr;
      gap: 25px;
      padding: 45px 0;
    }

    .hero h1 {
      font-size: 45px;
      letter-spacing: -2px;
    }

    .hero-content > p {
      font-size: 13px;
    }

    .hero-benefits {
      gap: 9px;
      font-size: 10px;
    }

    .hero-decoration {
      min-height: 290px;
      margin: 0 8px;
    }

    .hero-circle {
      width: 205px;
      height: 205px;
    }

    .basket {
      font-size: 95px;
    }

    .hero-note {
      bottom: 12px;
      left: 8px;
      padding: 12px;
    }

    .benefits {
      grid-template-columns: 1fr;
      gap: 5px;
      padding: 15px;
    }

    .benefits article {
      padding: 8px;
    }

    .products-section,
    .cart-section {
      padding-top: 60px;
    }

    .section-heading {
      align-items: stretch;
      flex-direction: column;
      gap: 20px;
    }

    .section-heading h2,
    .cart-heading h2 {
      font-size: 30px;
    }

    .search-box {
      width: 100%;
    }

    .category-list {
      gap: 7px;
      margin: 22px 0;
    }

    .category-list button {
      padding: 9px 12px;
      font-size: 11px;
    }

    .product-grid {
      grid-template-columns: repeat(2, minmax(0, 1fr));
      gap: 12px;
    }

    .product-image {
      height: 145px;
    }

    .product-info {
      padding: 12px;
    }

    .product-info h3 {
      font-size: 13px;
    }

    .price {
      font-size: 14px;
    }

    .add-button {
      width: 34px;
      height: 34px;
    }

    .cart-content {
      grid-template-columns: 1fr;
    }

    .cart-item {
      gap: 10px;
      padding: 11px;
    }

    .cart-item-image {
      width: 55px;
      height: 55px;
    }

    .cart-item-info strong {
      font-size: 11px;
    }

    .cart-item-end strong {
      font-size: 11px;
    }

    .cart-item-end {
      max-width: 85px;
    }

    .about-section {
      align-items: flex-start;
      flex-direction: column;
      gap: 15px;
      margin-bottom: 45px;
      padding: 25px;
    }

    .about-icon {
      width: 65px;
      height: 65px;
      font-size: 30px;
    }

    .about-section h2 {
      font-size: 25px;
    }

      footer {
      flex-direction: column;
      gap: 10px;
      text-align: center;
    }

  .contact-section {
      grid-template-columns: 1fr;
      gap: 22px;
      margin: 35px 0 55px;
      padding: 22px;
    }

    .contact-form {
      padding: 17px;
    }

    .auth-modal {
      padding: 27px;
    }
  }
</style>