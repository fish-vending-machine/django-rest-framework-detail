이 글을 보기 전에 아래의 내용들을 먼저 숙지하는 것이 좋다.

1. RESTful api
2. Object oriented programming
3. Serialize
4. Django model class

## 개요

---

- API web service 는 client의 특정 동작 중 database의 변경/열람 이 필요한 경우에 해당 요청을 받아 처리하는 프로그램이다. Database 의 변경 / 열람을 아래의 다섯 가지 요청 방식으로 나눔
    - List
    - Retrieve
    - Create
    - Update
    - Delete

    우리의 service는 database에서 가져온 데이터를 해당 언어의 객체로 가지고 있다. Request - response 는 약속된 규격이 있는데(XML, JSON, YAML 등의 양식), 이것은 javascript - JSON 이 아닐 경우 위에 언급한 객체와는 다른 형식일 수 밖에 없다.(이것이 Node.js만의 장점이기도 하다. JSON 규격을 사용하는 경우 serialize과정이 불필요하다.) 따라서 요청을 알아듣고 응답하기 위해서는  `web service 언어 객체` ↔ `약속한 규격` 사이의 변환이 필요함. 이 변환을 하는 객체를 serializer 라고 하고, 이 변환 작업을 serialization (언어 객체 → 약속한 규격), deserialization(약속한 규격 → 언어 객체) 이라고 한다.

- Response를 주기 위해서는 항상 serialization 과정이 필요하다. (데이터셋을 가져오는 list와 retrieve 외에도 create, update, delete 의 결과물을 약속된 규격으로 보여주기 위함)
- Deserialization은 request data 를 기반으로 데이터 객체를 생성할 시에만 필요하므로 create, update 시에만 필요하다.
- 결국 우리의 서비스는 요청을 받으면
    1. **해당 요청을 파싱하고,**
    2. **적절한 코드블럭에 넘겨준 후에 필요하면 deserialize 과정을 거쳐서 database에 저장하고,**
    3. **결과물을 database에서 가져와서,**
    4. **Serialize 하여 응답으로 돌려준다.**

    그럼 이제 위의 동작들을 기반으로 요청이 서비스에 inbound되는 시점에서 필요한 구조체들을 생각해보자.

    1. 요청한 url을 기반으로 실행할 코드를 찾아주는 router
    2. 요청받은 정보들(request data, method, query parameter, url, authentication info)을 parsing 하여 적절한 데이터 객체와 serializer를 가져와 응답을 해줄 수 있는 controller
    3. 필요한 경우 데이터 객체를 약속한 규격으로 변환해주는 serializer
    4. 데이터 객체를 db와 소통하여 save, load 할수 있는 model

    물론 세세하게 더 나누면 request parser, authenticater 등등 많은 구조체가 필요하겠지만 크게 나누면 누가 생각하던 위 네개의 class는 필요할 것이고, 당연하게도 DRF는 위 네 종류의 class를 준비해두었다.(models, serializers, views or viewsets, routers)

    이에 따라 어떤 코드를 추가, 수정한다고 할 때 위 네 가지 class 중 어디에 해당 코드를 삽입해야 하는 지 결정하는 것이 좋다.

- Client 의 요청 자체가 database 와의 소통을 위한 것(특정 값의 조회, 생성, 수정, 삭제)이라고 위에 언급한대로 결국에는 모든 요청들은 database와의 소통이 필요하기 때문에 controller 와 serializer 모두 model을 참조하고, 이 때문에 model class 는 전체 process 의 항상 가장 뒤쪽에 위치하게된다. 따라서 일반적으로 일관된 규칙을 가질 수 밖에 없는 business logic은 model에 위치하는 것이 재사용성 입장에서 가장 좋다.

## Model

---

Django에 관련된 내용이므로 pass

## Serializer

---

- Response를 돌려주기 위해서는 database에서 query된 결과물이건 요청을 통해 새로 생성된 결과물이건 항상 serialization 과정을 거쳐야 하며, 이 때문에 serializer의 동작 방식을 이해하는 것이 아주 중요하다.
DRF의 serializer는 크게 serialization, create, update 세 가지의 일을 수행하며, generic programming 방식으로 구현되어 있어서 argument의 종류에 따라 동작이 달라진다.
serializer는 instance, data, many 세 개의 argument를 기본으로 받는다.
    1. many

        instance argument 로 받을 instance 의 형식을 serializer에게 알려주는 데 사용한다. queryset을 instance로 받는 경우 true, model object 를 instance로 받는 경우 false이며, default value는 false이다.

    2. data

        create나 update 동작을 할 때만 받으며 생성하거나 수정할 정보를 이 argument를 통해 받는다. dictionary 형이며 keyword argument로 받는다. instance argument의 값이 있을 경우 update를, 없을 경우 create를 실행하며, 두 함수 모두 save method에서 알아서 argument를 parsing하여 실행된다.

    3. instance

        serialize, deserialize 를 할 대상을 받는다. instance가 있고 data가 있을 경우에 update가 실행되며, instance만 있고 data가 없는 경우 serialize 해서 JSON 형식의 데이터를 리턴한다.

    serialization 은 이미 database에 쓰기까지 완료된 데이터 객체를 JSON 형식으로 변환하는 것이므로, 이미 database에 기록되어있는 시점에서 valdiation이 완료됐다고 간주한다. 따라서 이 과정은 validation을 따로 하지 않는다.

    하지만 deserialization은 유저의 요청을 받아서 database 에 작성하는 과정이므로, 반드시 validation을 해야 하며, 이것은 serializer 코드 자체가 그러지 않으면 model class가 database와 소통하는 것을 막는다.

    코드로 보면 아래와 같다.

    ```python
    from django.db import models
    from rest_framework import serializers

    class Foo(models.Model):
        bar = models.IntegerField()

    class FooSerializer(serializers.ModelSerializer):
        class Meta:
            model = Foo
            fields = (
                'bar'
            )

    data = {
        "bar": 123
    }

    # list serialization
    serializer = FooSerializer(Foo.objects.all(), many=True)
    serializer.data

    # retrieve serialization
    serializer = FooSerializer(Foo.objects.last())
    serializer.data

    # create
    serializer = FooSerializer(data=data)
    serializer.is_valid()
    serializer.save()
    serializer.data

    # update
    serializer = FooSerializer(Foo.objects.last(), data=data)
    serializer.is_valid()
    serializer.save()
    serializer.data

    # delete 는 serializer를 거치지 않음 (serialize, deserialize를 할 필요가 전혀 없다.)
    Foo.objects.last().delete()
    ```

- is_valid method 는 들어온 데이터의 정합성을 검사하는 과정이며, 순서대로 아래 네 가지 방식을 통해 검사한다.
    1. 각 field의 required, allow_null, allow_blank, read_only check
    2. 기본적으로 정의된 각 field의 type과 data argument로 들어온 field 이름과 맞는 key에 포함된 value의 type이 맞는 지 검사
    3. 각 field의 validator 실행
    4. 전체 validation 실행

    이 내용은 Serializer class 의 run_validation 함수를 보면 확인이 가능한다.

    ```python
        def run_validation(self, data=empty):
            # 1
            (is_empty_value, data) = self.validate_empty_values(data)
            if is_empty_value:
                return data

            # 2, 3
            value = self.to_internal_value(data)
            try:
                self.run_validators(value)
                # 4
                value = self.validate(value)
                assert value is not None, '.validate() should return the validated data'
            except (ValidationError, DjangoValidationError) as exc:
                raise ValidationError(detail=as_serializer_error(exc))

            return value

        def validate_empty_values(self, data):
            if self.read_only:
                return (True, self.get_default())

            if data is empty:
                if getattr(self.root, 'partial', False):
                    raise SkipField()
                if self.required:
                    self.fail('required')
                return (True, self.get_default())

            if data is None:
                if not self.allow_null:
                    self.fail('null')
                elif self.source == '*':
                    return (False, None)
                return (True, None)

            return (False, data)

        def to_internal_value(self, data):
            if not isinstance(data, Mapping):
                message = self.error_messages['invalid'].format(
                    datatype=type(data).__name__
                )
                raise ValidationError({
                    api_settings.NON_FIELD_ERRORS_KEY: [message]
                }, code='invalid')

            ret = OrderedDict()
            errors = OrderedDict()
            fields = self._writable_fields

            for field in fields:
                validate_method = getattr(self, 'validate_' + field.field_name, None)
                primitive_value = field.get_value(data)
                try:
                    # 2
                    validated_value = field.run_validation(primitive_value)
                    if validate_method is not None:
                        # 3
                        validated_value = validate_method(validated_value)
                except ValidationError as exc:
                    errors[field.field_name] = exc.detail
                except DjangoValidationError as exc:
                    errors[field.field_name] = get_error_detail(exc)
                except SkipField:
                    pass
                else:
                    set_value(ret, field.source_attrs, validated_value)

            if errors:
                raise ValidationError(errors)

            return ret
    ```

    3번의 경우, serializer 내에 정의된 `validate_{field_name}` 함수들을 runtime 단계에서 모조리 취합해서 들어온 데이터의 각 key를 검사한다. 4번의 경우 serializer 내에 정의된 `validate` 함수를 1회 실행하는 것으로 검사한다. 즉 각 field에 대한 validation이 필요한 경우 validator를 정의하는 방식으로 코드를 작성하고, 전반적인 검사나 여러 field가 involve 되어있는 validation의 경우 validate 함수 내에 정의하는 방식으로 코드를 작성해야 한다는 의미이다.

- deserialization 시 serializer의 자세한 동작 방식은 아래와 같다.
    1. data argument 로 들어온 데이터를 constructor에서 self.initial_data에 저장.
    2. is_valid call 시 추가적으로 사용할 self._validated_data, self._errors 를 각각 dict로 선언. 각 attribute는 private indicator 를 뗀 getter 함수가 존재
    3. is_valid call을 하면 위에 설명한 validation이 진행되며, validation이 실패하면 self._errors에 에러 내용을 넣고 self._validated_data 를 빈 dict로, 모두 성공하면 self._validated_data 에 validation이 끝난 데이터를 넣음
    4. save call을 하면 validated_data에 들어있는 데이터를 사용해 데이터 객체를 생성, db와 소통하여 row를 기록함. 여기서 instance가 있으면  update를, 없으면 create를 실행함.
    5. serializer.data를 통해 생성된 데이터 객체의 정보를 serialize한 상태로 볼 수 있음.

    위에서 설명한 소스코드의 내용 중 assertion들을 제외하고 추리면 대략적으로 아래와 같다. (BaseSerializer는 Serializer의 parent이며, Serializer는 ModelSerializer 의 parent로 아래 작성된 method들을 변경없이 거의 대부분 공유한다.)

    ```python
    class BaseSerializer(Field):
        def __init__(self, instance=None, data=empty, **kwargs):
            self.instance = instance
            if data is not empty:
                # 1
                self.initial_data = data
            self.partial = kwargs.pop('partial', False)
            self._context = kwargs.pop('context', {})
            kwargs.pop('many', None)
            super().__init__(**kwargs)

        def is_valid(self, raise_exception=False):
            # 2
            if not hasattr(self, '_validated_data'):
                try:
                    # 3
                    self._validated_data = self.run_validation(self.initial_data)
                except ValidationError as exc:
                    self._validated_data = {}
                    self._errors = exc.detail
                else:
                    self._errors = {}

            if self._errors and raise_exception:
                raise ValidationError(self.errors)

            return not bool(self._errors)

        # 2
        @property
        def errors(self):
            if not hasattr(self, '_errors'):
                msg = 'You must call `.is_valid()` before accessing `.errors`.'
                raise AssertionError(msg)
            return self._errors

        # 2
        @property
        def validated_data(self):
            if not hasattr(self, '_validated_data'):
                msg = 'You must call `.is_valid()` before accessing `.validated_data`.'
                raise AssertionError(msg)
            return self._validated_data

        # 4
        def save(self, **kwargs):
            assert hasattr(self, '_errors'), (
                'You must call `.is_valid()` before calling `.save()`.'
            )

            assert not self.errors, (
                'You cannot call `.save()` on a serializer with invalid data.'
            )

            assert not hasattr(self, '_data'), (
                "You cannot call `.save()` after accessing `serializer.data`."
                "If you need to access data before committing to the database then "
                "inspect 'serializer.validated_data' instead. "
            )

            validated_data = dict(
                list(self.validated_data.items()) +
                list(kwargs.items())
            )

            if self.instance is not None:
                self.instance = self.update(self.instance, validated_data)
            else:
                self.instance = self.create(validated_data)

            return self.instance

        # 5
        @property
        def data(self):
            if not hasattr(self, '_data'):
                if self.instance is not None and not getattr(self, '_errors', None):
                    self._data = self.to_representation(self.instance)
                elif hasattr(self, '_validated_data') and not getattr(self, '_errors', None):
                    self._data = self.to_representation(self.validated_data)
                else:
                    self._data = self.get_initial()
            return self._data

    ```

    save method의 assertion 일부를 남겨둔 이유는 실제 save call 시에 어떤 검사들이 들어가는 지 보여주기 위함이다.

    1. self._errors는 is_valid call 시에 선언되는 attribute 이므로 save call 전에 is_valid call이 되었는지 검사할 수 있는 요소가 된다.
    2. python 에서 `not {} == True` 이므로 self.errors가 빈 dictionary 이면 validation이 성공했다는 것을 확인할 수 있는 요소가 된다.
    3. to_representation은 메모리를 꽤 많이 사용하는 동작(instance field 중 serializer에 정의된 field들을 JSON 형식으로 변경하는 serialize가 여기서 일어난다)이므로 변경이 있지 않은 이상 한번 cache한 후에 재사용을 하는것이 좋은데, 이 때문에 `self.data` call 시에 해당 결과물을 self._data에 저장하여 사용한다. 이 때 update같은 경우는 save call을 먼저 하지 않으면 기존의 instance를 serialize해서 보여주는데, 이것은 save call을 하고나면 변하는 값이고, 이렇게 되면 두번의 to_repr 동작이 일어나게 되므로 이것을 원천적으로 막기위해서 save 시에 to_repr가 불려왔는 지 self._data의 존재 유무로 판단을 하고, 불렸다면 save를 막는다.

    소스코드를 보면 위에서 말한 to_repr 말고도 보다시피 serializer는 내부적으로 아주 많은 데이터 구조를 생성하고 상당수의 데이터들은 서로 거의 중복되는 값을 가지고 있다. 그래서 DRF의 단점으로 무겁다(메모리 사용량이 많다) 는 단점이 꼽힌다. 하지만 이로 인해 아주 간편하고 다이나믹한 api를 제공해서 실제 개발자가 작성해야 하는 코드를 많이 줄여준다.

## Viewset

---

DRF는 3종류의 controller를 지원한다.

1. Function based view
2. Class based APIView (Django의 View class를 상속받아 만들어짐)
3. Class based ViewSet (APIView를 상속받아 만들어진 view의 set)

어떤 종류의 controller를 사용하던 기본적으로 html template renderer(브라우저로 api 주소 접속 시 보이는 화면)와 request body parser(json, form, multipart, file 네 종류)를 지원해주며, 설정에 따라 controller가 필요로 하는 기능들(throttle, authentication, permission, content negotiation 등)을 지원한다.

Production level에서는 유지보수가 용이한 ViewSet, 그 중에서도 model instance들과 serializer class를 다루는 GenericViewSet을 주로 사용하고 이것은 GenericAPIView를 상속받은 class이므로 함께 설명하고 function based view에 대한 설명은 넘긴다.

Viewset은 같은 자원을 공유하는 view들의 collection을 하나의 class로 나타낸 것으로, view class와 ViewSetMixin을 상속받은 class이다. router에 등록 시 특정 함수들 만을 적절한 url로 매핑한 api로 자동으로 생성하며, 이런 magic이 ViewSetMixin 내에서 일어난다. 자동으로 등록되는 특정 함수들은 아래와 같다.

- list (not detail)
- create (not detail)
- retrieve (detail)
- update (detail)
- delete (detail)
- functions with action decorator

LCRUD는 not detail인 경우 viewset이 등록 된 url root로 url이 생성되며 detail인 경우 url root 뒤에 `/{lookup kwarg}` 로 api가 생성된다.

Action decorated function는 url_path를 따로 지정하지 않으면 not detail인 경우 url root 뒤에 해당 함수의 이름이 붙은 형식으로 api가 등록되며 detail인 경우 `url_root/{lookup kwarg}/{함수이름}` 의 형식으로 등록된다.

모든 api function들은 request 를 두 번째 인자로 받으며, detail인 경우 lookup keyword argument를 세 번째 인자로 받는다. 각 api들의 간단한 동작 구현이 `rest_framework.mixins` 내의 mixin class들로 구현되어 있으니, 이 내용들을 살펴보는 것이 좋다.

Viewset은 queryset, serializer_class 두 attribute를 override 하는 것을 기본으로 하며, 이 자원들을 view들이 공유하게 된다. 각 자원은 내장 함수들을 통해 관리해야 하며, 아래와 같다.

- queryset
    - get_queryset(self)
    - get_object(self)
- serializer_class
    - get_serializer_class(self)
    - get_serializer_context(self)
    - get_serializer(self)

위에 설명한 대로 실제 View의 동작은 GenericAPIView 안에서 일어나며, 위 함수들을 소스코드에서 살펴보면 아래와 같다.

```python
class GenericAPIView(views.APIView):
    queryset = None
    serializer_class = None

    lookup_field = 'pk'
    lookup_url_kwarg = None

    def get_queryset(self):
        queryset = self.queryset
        if isinstance(queryset, QuerySet):
            # Ensure queryset is re-evaluated on each request.
            queryset = queryset.all()
        return queryset

    def get_object(self):
        queryset = self.filter_queryset(self.get_queryset())

        # Perform the lookup filtering.
        lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field

        filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
        obj = get_object_or_404(queryset, **filter_kwargs)

        # May raise a permission denied
        self.check_object_permissions(self.request, obj)

        return obj

    def get_serializer(self, *args, **kwargs):
        serializer_class = self.get_serializer_class()
        kwargs['context'] = self.get_serializer_context()
        return serializer_class(*args, **kwargs)

    def get_serializer_class(self):
        return self.serializer_class

    def get_serializer_context(self):
        return {
            'request': self.request,
            'format': self.format_kwarg,
            'view': self
        }
```

여기서 중요한 것이 context인데, serializer 또한 하나의 객체이기 때문에 어떤 viewset에서 불려오는 지, 어떤 요청으로 불려왔는지에 따라 다른 동작을 해야할 경우가 있고, 이 때문에 context를 주입해주어야 한다. 물론 serializer를 instantiate 할 때

```python
serializer = FooSerializer(instance, context=self.get_serializer_context())
```

같이 불러와도 되나 이러면 코드에 중복이 계속 생기게 되므로 이보다는 get_serializer_class를 적절히 override하여 적당한 serializer class를 가져오게 만들고, get_serializer로 호출하는 것이 좋다.

전체적인 view의 동작은 더 상위 클래스인 APIView의 dispatch method 안에 구현되어 있고, 아래와 같이 동작한다.

1. WSGIRequest를 initialize_request를 통해 DRF Request object로 변환 (parser를 통해 request 구조 변환)
2. initial method에서 throttle, permission 등을 check
3. 요청받은 method에 맞춰 적절한 handler를 가져옴
4. handler를 통해 response를 생성 후 return

```python
class APIView(View):
    def dispatch(self, request, *args, **kwargs):
        self.args = args
        self.kwargs = kwargs
        # 1
        request = self.initialize_request(request, *args, **kwargs)
        self.request = request
        self.headers = self.default_response_headers  # deprecate?

        try:
            # 2
            self.initial(request, *args, **kwargs)

            # Get the appropriate handler method
            if request.method.lower() in self.http_method_names:
                # 3
                handler = getattr(self, request.method.lower(),
                                  self.http_method_not_allowed)
            else:
                handler = self.http_method_not_allowed

            # 4
            response = handler(request, *args, **kwargs)

        except Exception as exc:
            response = self.handle_exception(exc)

        self.response = self.finalize_response(request, response, *args, **kwargs)
        return self.response
```

여기서 handler가 우리가 작성한 view function들이다.

---

## Router

TODO
