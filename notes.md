# Notas sobre Django

## Init

Para empezar un nuevo proyecto:

``` bash
django-admin startproject <project_name>
```

Genera un directorio de proyecto con el nombre del proyecto y estructura:

``` bash
nombre_proyecto
│   manage.py
│
└───nombre_proyecto
        asgi.py
        settings.py
        urls.py
        wsgi.py
        __init__.py
```

* `manage.py`: Script de python para accesar las funciones de django-admin especificas al proyecto.
* `settings.py`: Script de python con la configuracion del proyecto.
* `urls.py`: Script de python con patrones URL del servidor.
* `asgi.py` y `wsgi.py`: Interfaces ASGI (Asynchronous Server Gateway Interface) y WSGI (Web Server Gateway Interfaces) para la comunicacion entre el servidor web y Python.
* `__init__.py`: Script de python para agregar opciones extra de inicializacion

## Servidor de prueba y aplicaciones

Para levantar el proyecto en un servidor de prueba, utilizamos el comando:

``` bash
python manage.py runserver
```

Por defecto, Django levanta un administrador web para el proyecto bajo el URL `admin/`. Tambien levanta aplicaciones para la autentificacion de usuarios, sesiones, mensajes, contenido estatico y para tipos de contenido. wsvincent tiene una [lista en GitHub](https://github.com/wsvincent/awesome-django) cuidada de aplicaciones populares para proyectos de Django.

En lugar de generar todo el proyecto bajo un mismo modulo, Django divide sus proyectos en aplicaciones, las cuales pueden ser reutilizadas en proyectos distintos. Para crear una aplicacion, utilizamos el comando:

``` bash
python manage.py startapp <nombre_aplicacion>
```

Esto genera la estructura:

``` bash
nombre_aplicacion
│   admin.py
│   apps.py
│   models.py
│   tests.py
│   views.py
│   __init__.py
│
└───migrations
        __init__.py
```

* `admin.py`: Registro de modelos para gestionar con el administrador web de Django
* `apps.py`: Archivo para dar de alta las aplicaciones como clases de python.
* `models.py`: Archivo para crear modelos para el [ORM de Django](#ORM-de-Django).
* `tests.py`: ?
* `views.py`: Archivo para determinar las respuestas del servidor.
* `migrations`: Directorio para los scripts autogenerados por Django para generar cambios en la base de datos.

Para dar de alta la aplicacion dentro de nuestro proyecto, primero hay que crear un archivo urls.py que describa las rutas de la aplicacion, conteniendo la ruta URL, la funcion a invocar cuando se reciba un request, y opcionalemente un nombre para la ruta.

``` python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.home, name='blog-home'),
    path('about/', views.about, name='blog-about'),
]
```

Estas rutas son las rutas internas de la aplicacion. Para agregar la aplicacion al proyecto, primero hay que agregar la clase de la aplicacion (creada por default en `apps.py`) a `settings.py` en la variable `INSTALLED_APPS`

``` python
INSTALLED_APPS = [
    'blog.apps.BlogConfig',
    ...
]
```

Luego, hay que asignar las URL que queremos que la aplicacion use en `urls.py` (Nota, utilizar `<nombre_proyecto>/urls.py`, no `<nombre_aplicacion>/urls.py`):

``` python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls')),
]
```

En este caso, se agrega la aplicacion _blog_ al servidor bajo la ruta `blog/`, y sobre esta ruta se montan las distintas trayectorias que se dieron de alta en `<nombre_aplicacion>/urls.py`. Por ejemplo, `blog/` redirige hacia la respuesta creada por la funcion `blog.views.home`, y `blog/about` redirige hacia `blog.views.about`. Mas info sobre URLs: <https://docs.djangoproject.com/en/3.0/ref/urls/>.

## Templates

* `{% for x in y %}`
* `{% if x %} ... {% else %} .. {% endif %}`
* `{% url 'nombre_url' %}`
* `{% load static %}` y `{% static 'contenido_estatico' %}`
* `{% block nombre_bloque %} ... {% endblock nombre_bloque %}`
* `{% extends "nombre_template" %}`
* `{{varname}}`

## Forms

Django tiene algunas clases para representar forms como objetos de Python por defecto, incluyendo utilidades para ligar los forms a objetos del ORM. Django utiliza un token CSRF (_Cross Site Request Forgery_) para todas sus formulas. Django incluye algunas forms comunes por default, y tambien permite expandir estas heredandolas a una nueva clase, por ejemplo:

``` python
from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm

class UserRegisterForm(UserCreationForm):   #Hereda de la forma UserCreationForm de Django
    email = forms.EmailField()  # Agrega un campo para el correo del usuario

    class Meta:                 # Clase adicional para algunos metadatos
        model = User            # Tipo de modelo relacionado a la forma, en caso de querer utilizar la forma para crear cambios en la bd
        fields = ['username', 'email', 'password1', 'password2']    # Campos a desplegar
```

Un paquete popular para crear y decorar forms es [Django Crispy Forms](https://django-crispy-forms.readthedocs.io/en/latest/). Solo hay que instalar el paquete con:

``` bash
pip install django-crispy-forms
```

Y luego modificar `settings.py` para agregar _crispy forms_ a nuestro proyecto:

``` python
INSTALLED_APPS = [
    ...
    'crispy_forms',
    ...
]

...

CRISPY_TEMPLATE_PACK = '<framework de CSS utilizado>'
```

Luego, ,para estilizar correctamente un form, simplemente lo filtramos en el template, con el comando:

``` Django
{% load crispy_forms_tags %}
...
{{ form | crispy }}
```

## Admin y Auth

Por defecto, Django tiene herramientas para administrar la aplicacion y para la autentificacion de usuarios. Para utilizar el administrador, primero hay que asegurarse de que exista la base de datos para la autentificacion de usuarios (Ver [ORM](#orm-de-django)). Despues, solo hay que ejecutar el comando:

``` bash
python manage.py createsuperuser
```

Finalmente, bajo la ruta `admin/` por defecto se encuentra la interfaz web del administrador, a la cual se puede accesar con las credenciales creadas.

Para hacer la autentificacion de uusuarios, podemos utilizar la funcion [`authenticate`](https://docs.djangoproject.com/en/3.0/topics/auth/default/) de `django.contrib.auth`:

``` python
from django.contrib.auth import authenticate, login

def my_view(request):
    username = request.POST['username']
    password = request.POST['password']
    user = authenticate(request, username=username, password=password)
    if user is not None:
        login(request, user)
        # Redirect to a success page.
        ...
    else:
        # Return an 'invalid login' error message.
        ...
```

Similarmente, el logout de una sesion simplemente se hace con la funcion `logout` como se muiestra aqui:

``` python
from django.contrib.auth import logout

def logout_view(request):
    logout(request)
    # Redirect to a success page.
```

Django almacena la sesion del usuario bajo dentro de los requests HTTP del sitio. Esto permite acceso a ciertos datos sobre el usuario dentro del request, como:

* `request.user.username`
* `request.user.email`
* `request.user.is_authenticated`
* `request.user.is_staff`

Todos estos datos tambien pueden ser utilizados dentro de las plantillas sin necesidad de agregarlos a un contexto. Por defecto, los usuarios siguen la estructura del modelo [User](https://docs.djangoproject.com/en/3.0/ref/contrib/auth/#user-model) de Django.

Utilizando esta informacion, se pueden agregar redirects a los views para restringir el acceso a paginas unicamente a usuarios autenticados. Por ejemplo:

``` python
from django.conf import settings
from django.shortcuts import redirect

def my_view(request):
    if not request.user.is_authenticated:
        return redirect('%s?next=%s' % (settings.LOGIN_URL, request.path))
    # ...
```

De otro modo, podemos utilizar uno de los _decorators_ que Django aporta para agregar la funcionalidad deseada a alguna vista:

``` python
from django.contrib.auth.decorators import login_required

@login_required
def my_view(request):
    ...
```

Django tambien aporta algunas vistas preestablecidas para manejar la funcionalidad de login y logout:

``` python
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    ...
    path('login/', auth_views.LoginView.as_view(), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    ...
]
```

Por defecto, `LoginView` busca la plantilla para la vista bajo `registration/login.html`. Podemos cambiar esto utilizando el parametro `template_name='<login_template>.html'` en la funcion `as_view` (Idem para `Logout View`).

Documentacion a fondo de como personalizar aspectos de la autentificacion de usuarios: <https://docs.djangoproject.com/en/3.0/topics/auth/customizing/>

## ORM de Django

Django utiliza modelos para representar objetos de la s bases de datos. Estos modelos se crean bajo los archivos `models.py` de cada aplicacion. Deben de heredar de `django.db.models.Model`. Ejemplo:

``` python
from django.utils import timezone
from django.db import models
from django.contrib.auth.models import User


class Post(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    date_posted = models.DateTimeField(default=timezone.now)
    author = models.ForeignKey(User, on_delete=models.CASCADE)

```

* `models.__Field`: Clases para los distintos tipos de atributos que existen en la base de datos.
* `models.ForeignKey`: Llaves externas.

Por defecto, Django tiene tablas basicas para la autentificacion de usuarios en `django.contrib.auth.models`. Para actualizar la base de datos (Dada de alta en `<nombre_proyecto>/settings.py`, por defecto `db.sqlite3`), Django utiliza migraciones. Estas migraciones son scripts de python autogenerados que se encargan de crea las instrucciones SQL que ejecutaran. Una vez creados los scripts, estos son ejecutados, creando los cambios deseados. Para crear las migraciones, utilizamos el comando:

``` bash
python manage.py makemigrations
```

Esto crea nuevos scripts bajo los directorios _migrations_ de cada aplicacion. Para revisar las instrucciones SQL que cada script genera, utilizamos el comando:

``` bash
python manage.py sqlmigrate <nombre_aplicacion> <numero_migracion>
```

Finalmente, para ejecutar los cambios corremos el comando:

``` bash
python manage.py migrate
```

Para probar el uso de los modelos de Django se puede utilizar una terminal interactiva corriendo el comando:

``` bash
python manage.py shell
```

Dentro de esta terminal, podemos importar los modelos que queremos probar y hacer distintas consultas, por ejemplo:

``` python
from blog.models import Post
from django.contrib.auth.models import User

User.objects.all()                      # Regresa todos los usuarios
User.objects.first()                    # Regresa el primer usuario
User.objects.last()                     # Regresa el ultimo usuario
User.objects.filter(username='admin')   # Filtra los objetos
usr = User.objects.get(id=1)            # Guarda en usr el usuario con id 1

post_1 = Post(title='Blog 1',
              content='First Post Content!',
              author=usr)               # Crea un nuevo Post con los datos dados
post_1.save()                           # Guarda el nuevo post en la base de datos
Post.objects.filter(author=usr)         # Busca los posts en la base de datos del usuario usr
usr.post_set.all()                      # Otra manera de conseguir los posts del usuario usr
usr.post_set.create(title='Blog 2',
                    content='Second Post Content!',
                    author=usr)         # Otra manera de agregar un post del usuario usr

```

Otras funciones:

* `model.delete()`: Borra el objeto de la base de datos.

Tambien podemos utilizar la interfaz web del administrador de Django para gestionar tablas de la base de datos. Para esto, primero hay que agregar los modelos relevantes al archivo `<nombre_aplicacion>/admin.py` de la siguiente manera:

``` python
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

## Signals

Funcionalidad de Django que permite ejecutar codigo en respuesta a ciertos eventos. Por ejemplo, podemos agregar una señal de tal modo que cada vez que se cree un nuevo usuario, creemos un perfil para este, o que cuando un usuario actualiza su informacion, tambien se actualize su perfil:

``` python
from django.db.models.signals import post_save
from django.contrib.auth.models import User
from django.dispatch import receiver
from .models import Profile

@reciever(post_save, sender=User)
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)

@reciever(post_save, sender=User)
def save_profile(sender, instance, **kwargs):
    instance.profile.save()
```

## Class Based Views

En lugar de definir funciones para manejar las solicitudes HTTP, Django permite la creacion de clases mas robustas para amarrar logica de back-end a las vistas de la aplicacion. Django incluye varios tipos de class based views basados en funcionalidad tipica de paginas web. Por ejemplo, hay `ListViews`, `CreateViews`, `UpdateViews` y [mas](https://docs.djangoproject.com/en/3.0/topics/class-based-views/intro/).

Ejemplo de ListView:

``` python
# Como una funcion:
def home(request):
    context = {
        'posts': Post.objects.all()
    }
    return render(request, 'blog/home.html', context)

#Como un class based view
class PostListView(ListView):
    model = Post
    # Por defecto busca el template <app>/<model>_<viewtype>.html
    template_name = 'blog/home.html'
    # Nombre a asignar la lista de objetos dentro del template de django
    context_object_name = 'posts'
    # Ordenar mas nuevos primero
    context_object_name = 'posts'
    # Numero de elementos por paginas
    paginate_by = 5
```

Class badsed views tambien te permiten utilizar diferentes funciones para manejar distiontos tipos de requests al mismo URL (e.g. POST o GET) sin necesidad de utilizar `if` dentro de la funcion de respuesta para distinguir entre estos.

Otros ejemplos:

``` python
class PostDetailView(DetailView):
    model = Post


class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    fields = ['title', 'content']

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)

class PostDeleteView(LoginRequiredMixin, UserPassesTestMixin, DeleteView):
    model = Post
    success_url = 'blog-home'

    def test_func(self):
        post = self.get_object()
        return post.author == self.request.user
```
