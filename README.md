## JODY

JsOn for DYnamic sites

Rails приложения прекрасно работают с традиционным способом формирования статичных страниц. Суть этого процесса заключается в следующем:

1) пользователь нажимает на ссылку
2) запрос уходит на сервер
3) сервер обрабатывает запрос и рисует страницу
4) страница отправляется клиенту
5) браузер перерисовывает страницу

Этот способ прекрасен и понятен - так работал, работает и будет продолжать работать Интернет.

И вероятно все было бы прекрасно, если бы кол-во информации на одной странице многих проектов не стало слишком большим и полная перезагрузка страницы для выполнения простого действия не стала неоправданной роскошью

**Вот пример:**

На сайте с большим кол-вом "тяжолого" контента планируется реализовать следующую задачу:

1) Вход пользователя в систему должен быть выполнен без перезагрузки страницы
2) В случае успеха или ошибки должно появится уведомление
3) В случае успеха в боковой панели должен появится блок с информацией о счете пользователя и прочей информацией

### Вариант 1 - "классический"

Один из вариантов решения такой задачи заключается в определении формы входа, как формы посылающей AJAX запрос ```remote: true```, а в качестве ответа мы (вероятно) захотим получить управляющий JS код. Для этого в качестве типа ответа пришедшего с сервера мы укажем ```data: { type: :script } ```

Вот так может выглядеть форма входа

```ruby
= form_for(User.new, url: new_user_session_path, remote: true, data: { type: :script }) do |f|
  = f.text_field :email
  = f.password_field :password
```

Вот так может выглядеть метод контроллера Session

```ruby
def create
  if @user = User.find_by_email_and_password(params[:email], params[:password])
    sign_in @user
    render template: :sign_in_success, layout: false
  else
    render template: :sign_in_error, layout: false
  end
end
```

и примерно вот так будет выглядеть один из шаблонов:

**sign_in_success.html.erb**

```erb
$("#sidebar").append("<%= render partial: "user_accaunt_block" %>")
App.sidebar_initialize()

$("#login_block").empty().append("<%= render partial: "user_info" %>")
App.user_info_initialize()

AppNotifier.alert("<%= t ".login_success" %>")
```

Такой вариант решения задачи более чем распространен и используется во многих проектах.

Однако многие могут заметить, что мы начали писать код стороны клиента на стороне сервера. И как минимум это грозит нам тем, что наш JQuery код в скором времени займет не только положенное ему место в каталоге **app/assets/javascript**, но и половину нашего основного приложения, что не может не заставить задуматься о других вариантах решения задачи.

### Вариант 2 - JODY

Я был чрезвычайно обрадован, когда увидел в одном из проектов фрагмент кода, который был очень похож на то, о чем я некоторое время назад размышлял. Несколько доработав подход я готов вынести основную идею техники на обсуждение.

**Основные положения техники:**

1) Мы не должны писать JS код на стороне сервера и засорять наши прекрасные ruby файлы (даже если это ERB файлы)
2) Шаблоны и фрагменты HTML мы можем переавать с сервера используя JSON как структурированное хранилище различных элементов
3) JSON может содеражть управляющие инструкции для клиентской стороны (например, имена функций, которые требется выполнить после получения ответа от сервера)

**Основными элементами в технике JODY являются**

1) JS посредник между серверной и клиентской стороной
2) JBuilder - для формирования JSON структур

Говоря о посреднике между серверной частью и клиентом я говорю о файле где выполняется определение глобальной реакции на AJAX ответы:

Это может выглядеть примерно так:

```coffeescript
$(document).ajaxError (event, request, settings) ->
  if typeof (data = request.responseJSON) is "object"
    flash  = data.flash
    errors = data.errors

    TheNotification.show_flash flash
    TheNotification.show_errors errors
    
    Redirect.processor data

$(document).ajaxSuccess (event, xhr, params, data) ->
  flash  = data.flash
  errors = data.errors
  html_partials = data.html_partials
  initializers  = data.initializers

  TheNotification.show_flash flash
  TheNotification.show_errors errors
  
  #...
  
  HtmlRenderer.render html_partials
  InvokeFunctions.exec initializers
  
  Redirect.processor data
```

Идея заключается в том, что  принимая в одной точке своего приложения структурированные ответы и используя ряд соглашений, вы можете избавить своих разработчиков от написания огромного количества кода, который только и занимается тем, что выполняет рутинны операции вида - **показать уведомление**, **вставить полученный HTML в DOM**, **выполнить функции инициализации HTML** и прочее.

При наличии такого посредника вы сможете избавится от огромного кол-ва рутинных задач и при этом уникальную логику обработки AJAX запросов вы все так же сможете реализовать с помощью добавления обработчиков на конкретные элемены

я говорю о чем-то таком

```coffeescript
$('#login_form').on 'ajax:success', (event, xhr, status) ->
  # Что-то действительно особенное, после того, как произошел логин
```

Все что вам осталось - это начать использовать JBuilder что бы аккуратно формировать ответы сервера с заданной стурктурой.

Вернемся начальной задаче и посмотрим, как бы она могла быть решена с использованием JODY


Вот так может выглядеть форма входа для JODY

```ruby
= form_for(User.new, url: new_user_session_path, remote: true, data: { type: :json }) do |f|
  = f.text_field :email
  = f.password_field :password
```

Вот так может выглядеть метод контроллера Session

```ruby
def create
  if @user = User.find_by_email_and_password(params[:email], params[:password])
    sign_in @user
  end
end
```

и примерно вот так будет выглядеть jbuilder шаблон:

**create.json.jbuilder**

```ruby
if current_user
  json.flash do
    json.info t '.login_success'
  end

  json.html_partials do |html|
    html.sidebar     render partial: "user_accaunt_block.html"
    html.login_block render partial: "user_info.html"
  end
  
  json.invoke do |function|
    function.sidebar_initialize
    function.user_info_initialize
  end
else
  json.flash do
    json.error t ".login_failure"
  end
end
```

Глядя на этот простой пример вы можете отметить, что количество кода вьюшки увеличилось по сравнению с исходным, но, оценивая полученный результат не забудьте отметить и следующие моменты

1) backend разработчик полностью отделен от особенностей реализации forntend части, и вероятно большинство задач не потребует от него дополнительных навыков
2) Благодаря соглашениям вы наверняка сократите кол-во JS кода на клиентской стороне, который реализует множество рутинных операций
3) Вы очистили свои вьюшки от JS вставок и существенно улучшили поддерживаемость своей системы

#### Благодарности

[@milushov](https://github.com/milushov) - за то, что своим кодом дал мне повод думать, что одна из моих идей может успешно работать
