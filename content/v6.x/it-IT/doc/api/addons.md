# Addons C++

<!--introduced_in=v0.10.0-->

Gli Addons di Node.js sono oggetti condivisi collegati in modo dinamico, scritti in C++, che possono essere caricati all'interno di Node.js usando la funzione [`require()`](globals.html#globals_require), ed utilizzati come se fossero un normale modulo di Node.js. Essi vengono utilizzati principalmente per fornire un'interfaccia tra JavaScript in esecuzione in Node.js e le librerie C/C++.

Al momento, il metodo per implementare gli Addons è piuttosto complicato, coinvolgendo la conoscenza di diversi componenti e diversi API :

* V8: la libreria C++ che Node.js utilizza attualmente per fornire l'implementazione JavaScript. V8 fornisce i meccanismi per creare oggetti, chiamare funzioni, ecc. L'API di V8 è documentata principalmente nel file di intestazione `v8.h` (`deps/v8/include/v8.h` nell'albero sorgente di Node.js), che è anche disponibile [online](https://v8docs.nodesource.com/).

* [libuv](https://github.com/libuv/libuv): La libreria C che implementa il ciclo di eventi Node.js, i suoi thread di lavoro e tutti i comportamenti asincroni della piattaforma. Inoltre, funge da libreria di astrazione multipiattaforma, dando un facile accesso di tipo POSIX, su tutti i principali sistemi operativi, a molte attività di sistema comuni, come l'interazione con il filesystem, i socket, i timer e gli eventi di sistema. libuv fornisce anche un'astrazione di thread di tipo pthreads (POSIX-threads) che può essere usata per alimentare gli Addons asincroni più sofisticati che devono andare oltre il ciclo degli eventi standard. Gli autori degli Addon sono incoraggiati a pensare a come evitare il blocco del ciclo degli eventi con I/O oppure con altre attività che richiedono molto tempo nello scaricare il lavoro tramite libuv in operazioni di sistema non-blocking, threads di lavoro od un uso personalizzato dei threads di libuv.

* Librerie interne di Node.js. Node.js stesso esporta un numero di API C++ che gli Addons possono utilizzare &mdash; la più importante delle quali è la classe `node::ObjectWrap`.

* Node.js include una serie di altre librerie collegate in modo statico, tra cui OpenSSL. Queste altre librerie si trovano nella directory `deps/` all'interno dell'albero sorgente (source tree) di Node.js. Solo i simboli di V8 ed OpenSSL vengono appositamente ri-esportati da Node.js e possono essere utilizzati in varie estensioni dagli Addons. Vedi [Collegamento alle dipendenze di Node.js](#addons_linking_to_node_js_own_dependencies) per ulteriori informazioni.

Tutti i seguenti esempi sono disponibili per il [download](https://github.com/nodejs/node-addon-examples) e possono essere utilizzati come punto di partenza per il proprio Addon.

## Hello world

Questo esempio di "Hello world" è un semplice Addon, scritto in C++, che è l'equivalente del seguente codice JavaScript:

```js
module.exports.hello = () => 'world';
```

Prima di tutto, crea il file `hello.cc`:

```cpp
// hello.cc
#include <node.h>

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void Method(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  args.GetReturnValue().Set(String::NewFromUtf8(isolate, "world"));
}

void init(Local<Object> exports) {
  NODE_SET_METHOD(exports, "hello", Method);
}

NODE_MODULE(addon, init)

}  // namespace demo
```

Si noti che tutti gli Addons di Node.js devono esportare una funzione di inizializzazione seguendo il modello:

```cpp
void Initialize(Local<Object> exports);
NODE_MODULE(module_name, Initialize)
```

Non c'è alcun punto e virgola dopo `NODE_MODULE` poiché non è una funzione (vedi `node.h`).

Il `module_name` deve corrispondere al filename del file binario finale (escludendo il suffisso .node).

Quindi nell'esempio `hello.cc`, la funzione di inizializzazione è `init` ed il module name dell'Addon è `addon`.

### Building

Una volta che è stato scritto il codice sorgente, esso dev'essere compilato nel file binario `addon.node`. Per fare ciò, crea un file chiamato `binding.gyp` nel primo livello del progetto che descriva la configurazione del build del tuo modulo usando un formato di tipo JSON. Questo file è usato da [node-gyp](https://github.com/nodejs/node-gyp) — uno strumento scritto appositamente per compilare gli Addons di Node.js.

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "hello.cc" ]
    }
  ]
}
```

*Nota: Una versione dell'utility `node-gyp` è in bundle e distribuita con Node.js come parte di `npm`. Questa versione non è resa direttamente disponibile per gli sviluppatori ed è pensata solo per supportare la possibilità di utilizzare il comando `npm install` per compilare ed installare gli Addons. Gli sviluppatori che desiderano utilizzare `node-gyp` direttamente possono installarlo tramite il comando `npm install -g node-gyp`. Vedi le [istruzioni di installazione](https://github.com/nodejs/node-gyp#installation) di `node-gyp` per ulteriori informazioni, compresi i requisiti specifici della piattaforma.*

Una volta creato il file `binding.gyp`, utilizzare `node-gyp configure` per generare gli appropriati build files del progetto per la piattaforma corrente. Questo genererà un `Makefile` (su piattaforme Unix) oppure un file `vcxproj` (su Windows) nella directory `build/`.

Successivamente, invoca il comando `node-gyp build` per generare il file compilato `addon.node`. Questo verrà inserito nella directory `build/Release/`.

Quando si utilizza `npm install` per installare un Addon di Node.js, npm utilizza la propria versione in bundle di `node-gyp` per eseguire lo stesso insieme di azioni, generando una versione compilata dell'Addon per la piattaforma dell'utente su richiesta.

Una volta costruito, l'Addon binario può essere utilizzato da Node.js puntando [`require()`](globals.html#globals_require) al modulo costruito `addon.node`:

```js
// hello.js
const addon = require('./build/Release/addon');

console.log(addon.hello());
// Prints: 'world'
```

Si prega di vedere gli esempi di seguito per ulteriori informazioni oppure <https://github.com/arturadib/node-qt> per un esempio in corso di produzione.

Poiché il percorso esatto dell'Addon binario compilato può variare a seconda di come viene compilato (ad esempio potrebbe essere in `./build/Debug/`), gli Addons possono utilizzare il [bindings](https://github.com/TooTallNate/node-bindings) package per caricare il modulo compilato.

Si noti che, mentre l'implementazione del `bindings` package è più sofisticata nel modo in cui individua i moduli Addon, sta usando essenzialmente un modello try-catch del tipo:

```js
try {
  return require('./build/Release/addon.node');
} catch (err) {
  return require('./build/Debug/addon.node');
}
```

### Collegamento alle dipendenze di Node.js

Node.js utilizza un numero di librerie collegate in modo statico come V8, libuv ed OpenSSL. Tutti gli Addons sono necessari per il collegamento a V8 ma possono anche essere collegati a qualsiasi altra dipendenza. In genere, questo è semplice come includere l'appropriato `#include <...>` le istruzioni (es. `#include <v8.h>`) e `node-gyp` individueranno automaticamente le intestazioni appropriate. Tuttavia, ci sono alcune avvertenze da tenere in considerazione:

* Quando viene eseguito `node-gyp`, esso rileverà la specifica versione di rilascio di Node.js e scaricherà il codice sorgente tarball completo oppure solo le intestazioni. Se il codice sorgente completo viene scaricato, gli Addons avranno accesso completo all'insieme di tutte le dipendenze di Node.js. Tuttavia, se vengono scaricate solo le intestazioni di Node.js, allora saranno disponibili solo i simboli esportati da Node.js.

* `node-gyp` può essere eseguito utilizzando il flag `--nodedir` che punta ad un'immagine sorgente Node.js locale. Usando questa opzione, l'Addon avrà accesso all'insieme di tutte le dipendenze.

### Caricamento degli Addons utilizzando require()

L'estensione del filename dell'Addon binario compilato è `.node` (al contrario di `.dll` oppure `.so`). La funzione [`require()`](globals.html#globals_require) viene scritta per cercare i file con l'estensione `.node` ed inizializzarli come librerie con collegamenti dinamici.

Quando si chiama [`require()`](globals.html#globals_require), solitamente l'estensione `.node` può essere omessa e Node.js continuerà a ritrovare ed inizializzare l'Addon. Un avvertimento, tuttavia, è che Node.js prima tenterà di individuare e caricare i moduli oppure i file JavaScript che capitano a condividere lo stesso nome di base. For instance, if there is a file `addon.js` in the same directory as the binary `addon.node`, then [`require('addon')`](globals.html#globals_require) will give precedence to the `addon.js` file and load it instead.

## Native Abstractions for Node.js

Each of the examples illustrated in this document make direct use of the Node.js and V8 APIs for implementing Addons. It is important to understand that the V8 API can, and has, changed dramatically from one V8 release to the next (and one major Node.js release to the next). With each change, Addons may need to be updated and recompiled in order to continue functioning. The Node.js release schedule is designed to minimize the frequency and impact of such changes but there is little that Node.js can do currently to ensure stability of the V8 APIs.

The [Native Abstractions for Node.js](https://github.com/nodejs/nan) (or `nan`) provide a set of tools that Addon developers are recommended to use to keep compatibility between past and future releases of V8 and Node.js. See the `nan` [examples](https://github.com/nodejs/nan/tree/master/examples/) for an illustration of how it can be used.

## N-API

> Stability: 1 - Experimental

N-API is an API for building native Addons. It is independent from the underlying JavaScript runtime (ex V8) and is maintained as part of Node.js itself. This API will be Application Binary Interface (ABI) stable across version of Node.js. It is intended to insulate Addons from changes in the underlying JavaScript engine and allow modules compiled for one version to run on later versions of Node.js without recompilation. Addons are built/packaged with the same approach/tools outlined in this document (node-gyp, etc.). The only difference is the set of APIs that are used by the native code. Instead of using the V8 or [Native Abstractions for Node.js](https://github.com/nodejs/nan) APIs, the functions available in the N-API are used.

The functions available and how to use them are documented in the section titled [C/C++ Addons - N-API](n-api.html).

## Addon examples

Following are some example Addons intended to help developers get started. The examples make use of the V8 APIs. Refer to the online [V8 reference](https://v8docs.nodesource.com/) for help with the various V8 calls, and V8's [Embedder's Guide](https://github.com/v8/v8/wiki/Embedder's%20Guide) for an explanation of several concepts used such as handles, scopes, function templates, etc.

Each of these examples using the following `binding.gyp` file:

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "addon.cc" ]
    }
  ]
}
```

In cases where there is more than one `.cc` file, simply add the additional filename to the `sources` array. For example:

```json
"sources": ["addon.cc", "myexample.cc"]
```

Once the `binding.gyp` file is ready, the example Addons can be configured and built using `node-gyp`:

```console
$ node-gyp configure build
```

### Function arguments

Addons will typically expose objects and functions that can be accessed from JavaScript running within Node.js. When functions are invoked from JavaScript, the input arguments and return value must be mapped to and from the C/C++ code.

The following example illustrates how to read function arguments passed from JavaScript and how to return a result:

```cpp
// addon.cc
#include <node.h>

namespace demo {

using v8::Exception;
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::String;
using v8::Value;

// This is the implementation of the "add" method
// Input arguments are passed using the
// const FunctionCallbackInfo<Value>& args struct
void Add(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  // Check the number of arguments passed.
  if (args.Length() < 2) {
    // Throw an Error that is passed back to JavaScript
    isolate->ThrowException(Exception::TypeError(
        String::NewFromUtf8(isolate, "Wrong number of arguments")));
    return;
  }

  // Check the argument types
  if (!args[0]->IsNumber() || !args[1]->IsNumber()) {
    isolate->ThrowException(Exception::TypeError(
        String::NewFromUtf8(isolate, "Wrong arguments")));
    return;
  }

  // Perform the operation
  double value = args[0]->NumberValue() + args[1]->NumberValue();
  Local<Number> num = Number::New(isolate, value);

  // Set the return value (using the passed in
  // FunctionCallbackInfo<Value>&)
  args.GetReturnValue().Set(num);
}

void Init(Local<Object> exports) {
  NODE_SET_METHOD(exports, "add", Add);
}

NODE_MODULE(addon, Init)

}  // namespace demo
```

Once compiled, the example Addon can be required and used from within Node.js:

```js
// test.js
const addon = require('./build/Release/addon');

console.log('This should be eight:', addon.add(3, 5));
```

### Callbacks

It is common practice within Addons to pass JavaScript functions to a C++ function and execute them from there. The following example illustrates how to invoke such callbacks:

```cpp
// addon.cc
#include <node.h>

namespace demo {

using v8::Function;
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Null;
using v8::Object;
using v8::String;
using v8::Value;

void RunCallback(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  Local<Function> cb = Local<Function>::Cast(args[0]);
  const unsigned argc = 1;
  Local<Value> argv[argc] = { String::NewFromUtf8(isolate, "hello world") };
  cb->Call(Null(isolate), argc, argv);
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", RunCallback);
}

NODE_MODULE(addon, Init)

}  // namespace demo
```

Note that this example uses a two-argument form of `Init()` that receives the full `module` object as the second argument. This allows the Addon to completely overwrite `exports` with a single function instead of adding the function as a property of `exports`.

To test it, run the following JavaScript:

```js
// test.js
const addon = require('./build/Release/addon');

addon((msg) => {
  console.log(msg);
// Prints: 'hello world'
});
```

Note that, in this example, the callback function is invoked synchronously.

### Object factory

Addons can create and return new objects from within a C++ function as illustrated in the following example. An object is created and returned with a property `msg` that echoes the string passed to `createObject()`:

```cpp
// addon.cc
#include <node.h>

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  Local<Object> obj = Object::New(isolate);
  obj->Set(String::NewFromUtf8(isolate, "msg"), args[0]->ToString());

  args.GetReturnValue().Set(obj);
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", CreateObject);
}

NODE_MODULE(addon, Init)

}  // namespace demo
```

Per testarlo in JavaScript:

```js
// test.js
const addon = require('./build/Release/addon');

const obj1 = addon('hello');
const obj2 = addon('world');
console.log(obj1.msg, obj2.msg);
// Prints: 'hello world'
```

### Funzione factory

Un altro scenario comune è la creazione di funzioni JavaScript che racchiudono le funzioni C++ e le restituiscono a JavaScript:

```cpp
// addon.cc
#include <node.h>

namespace demo {

using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void MyFunction(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  args.GetReturnValue().Set(String::NewFromUtf8(isolate, "hello world"));
}

void CreateFunction(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, MyFunction);
  Local<Function> fn = tpl->GetFunction();

  // omettere questo per renderlo anonimo
  fn->SetName(String::NewFromUtf8(isolate, "theFunction"));

  args.GetReturnValue().Set(fn);
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", CreateFunction);
}

NODE_MODULE(addon, Init)

}  // namespace demo
```

Per testare:

```js
// test.js
const addon = require('./build/Release/addon');

const fn = addon();
console.log(fn());
// Prints: 'hello world'
```

### Wrapping degli objects C++

È anche possibile eseguire il wrapping di oggetti/classi C++ in un modo che consenta la creazione di nuove istanze utilizzando l'operatore JavaScript `new`:

```cpp
// addon.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Local;
using v8::Object;

void InitAll(Local<Object> exports) {
  MyObject::Init(exports);
}

NODE_MODULE(addon, InitAll)

}  // namespace demo
```

Quindi, in `myobject.h`, la classe wrapper eredita da `node::ObjectWrap`:

```cpp
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Local<v8::Object> exports);

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void PlusOne(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```

In `myobject.cc`, implementa i vari metodi che devono essere esposti. Sotto, viene esposto il metodo `plusOne()` aggiungendolo al prototipo del constructor:

```cpp
// myobject.cc
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Local<Object> exports) {
  Isolate* isolate = exports->GetIsolate();

  // Prepara il template del constructor
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(isolate, "MyObject"));
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  // Prototipo
  NODE_SET_PROTOTYPE_METHOD(tpl, "plusOne", PlusOne);

  constructor.Reset(isolate, tpl->GetFunction());
  exports->Set(String::NewFromUtf8(isolate, "MyObject"),
               tpl->GetFunction());
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  if (args.IsConstructCall()) {
    // Invocato come constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ? 0 : args[0]->NumberValue();
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invocato come semplice funzione `MyObject(...)`, trasformare in construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Context> context = isolate->GetCurrentContext();
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Object> result =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(result);
  }
}

void MyObject::PlusOne(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  MyObject* obj = ObjectWrap::Unwrap<MyObject>(args.Holder());
  obj->value_ += 1;

  args.GetReturnValue().Set(Number::New(isolate, obj->value_));
}
```

Per compilare questo esempio, il file `myobject.cc` deve essere aggiunto a `binding.gyp`:

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [
        "addon.cc",
        "myobject.cc"
      ]
    }
  ]
}
```

Testalo con:

```js
// test.js
const addon = require('./build/Release/addon');

const obj = new addon.MyObject(10);
console.log(obj.plusOne());
// Stampa: 11
console.log(obj.plusOne());
// Stampa: 12
console.log(obj.plusOne());
// Stampa: 13
```

### Factory di wrapped objects

In alternativa, è possibile utilizzare un modello di factory per evitare la creazione esplicita di istanze di oggetti utilizzando l'operatore `new` di JavaScript:

```js
const obj = addon.createObject();
// invece di:
// const obj = new addon.Object();
```

Prima di tutto, il metodo `createObject()` è implementato in `addon.cc`:

```cpp
// addon.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  MyObject::NewInstance(args);
}

void InitAll(Local<Object> exports, Local<Object> module) {
  MyObject::Init(exports->GetIsolate());

  NODE_SET_METHOD(module, "exports", CreateObject);
}

NODE_MODULE(addon, InitAll)

}  // namespace demo
```

In `myobject.h`, il metodo statico `NewInstance()` viene aggiunto per gestire la creazione di un'istanza per l'oggetto. Questo metodo sostituisce l'uso di `new` in JavaScript:

```cpp
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Isolate* isolate);
  static void NewInstance(const v8::FunctionCallbackInfo<v8::Value>& args);

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void PlusOne(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```

L'implementazione in `myobject.cc` è simile all'esempio precedente:

```cpp
// myobject.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Isolate* isolate) {
  // Prepara il template del constructor
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(isolate, "MyObject"));
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  // Prototipo
  NODE_SET_PROTOTYPE_METHOD(tpl, "plusOne", PlusOne);

  constructor.Reset(isolate, tpl->GetFunction());
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  if (args.IsConstructCall()) {
    // Invocato come constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ? 0 : args[0]->NumberValue();
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invocato come semplice funzione `MyObject(...)`, trasformare in construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Context> context = isolate->GetCurrentContext();
    Local<Object> instance =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(instance);
  }
}

void MyObject::NewInstance(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  const unsigned argc = 1;
  Local<Value> argv[argc] = { args[0] };
  Local<Function> cons = Local<Function>::New(isolate, constructor);
  Local<Context> context = isolate->GetCurrentContext();
  Local<Object> instance =
      cons->NewInstance(context, argc, argv).ToLocalChecked();

  args.GetReturnValue().Set(instance);
}

void MyObject::PlusOne(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  MyObject* obj = ObjectWrap::Unwrap<MyObject>(args.Holder());
  obj->value_ += 1;

  args.GetReturnValue().Set(Number::New(isolate, obj->value_));
}

}  // namespace demo
```

Ancora una volta, per compilare questo esempio, il file `myobject.cc` deve essere aggiunto a `binding.gyp`:

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [
        "addon.cc",
        "myobject.cc"
      ]
    }
  ]
}
```

Testalo con:

```js
// test.js
const createObject = require('./build/Release/addon');

const obj = createObject(10);
console.log(obj.plusOne());
// Stampa: 11
console.log(obj.plusOne());
// Stampa: 12
console.log(obj.plusOne());
// Stampa: 13

const obj2 = createObject(20);
console.log(obj2.plusOne());
// Stampa: 21
console.log(obj2.plusOne());
// Stampa: 22
console.log(obj2.plusOne());
// Stampa: 23
```

### Riavvolgere gli wrapped objects

In addition to wrapping and returning C++ objects, it is possible to pass wrapped objects around by unwrapping them with the Node.js helper function `node::ObjectWrap::Unwrap`. I seguenti esempi mostrano una funzione `add()` che può prendere due oggetti `MyObject` come argomenti di input:

```cpp
// addon.cc
#include <node.h>
#include <node_object_wrap.h>
#include "myobject.h"

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  MyObject::NewInstance(args);
}

void Add(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  MyObject* obj1 = node::ObjectWrap::Unwrap<MyObject>(
      args[0]->ToObject());
  MyObject* obj2 = node::ObjectWrap::Unwrap<MyObject>(
      args[1]->ToObject());

  double sum = obj1->value() + obj2->value();
  args.GetReturnValue().Set(Number::New(isolate, sum));
}

void InitAll(Local<Object> exports) {
  MyObject::Init(exports->GetIsolate());

  NODE_SET_METHOD(exports, "createObject", CreateObject);
  NODE_SET_METHOD(exports, "add", Add);
}

NODE_MODULE(addon, InitAll)

}  // namespace demo
```

In `myobject.h`, viene aggiunto un nuovo metodo pubblico per consentire l'accesso ai valori privati dopo aver eseguito l'unwrapping dell'oggetto.

```cpp
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Isolate* isolate);
  static void NewInstance(const v8::FunctionCallbackInfo<v8::Value>& args);
  inline double value() const { return value_; }

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```

L'implementazione di `myobject.cc` è simile a prima:

```cpp
// myobject.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Isolate* isolate) {
  // Prepara il template del constructor
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(isolate, "MyObject"));
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  constructor.Reset(isolate, tpl->GetFunction());
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  if (args.IsConstructCall()) {
    // Invocato come constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ? 0 : args[0]->NumberValue();
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invocato come semplice funzione `MyObject(...)`, trasformare in construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Context> context = isolate->GetCurrentContext();
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Object> instance =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(instance);
  }
}

void MyObject::NewInstance(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  const unsigned argc = 1;
  Local<Value> argv[argc] = { args[0] };
  Local<Function> cons = Local<Function>::New(isolate, constructor);
  Local<Context> context = isolate->GetCurrentContext();
  Local<Object> instance =
      cons->NewInstance(context, argc, argv).ToLocalChecked();

  args.GetReturnValue().Set(instance);
}

}  // namespace demo
```

Testalo con:

```js
// test.js
const addon = require('./build/Release/addon');

const obj1 = addon.createObject(10);
const obj2 = addon.createObject(20);
const result = addon.add(obj1, obj2);

console.log(result);
// Stampa: 30
```

### AtExit hooks

Un hook "AtExit" è una funzione che viene invocata dopo la fine del ciclo di eventi di Node.js ma prima che la JavaScript VM sia terminata e prima che Node.js si arresti. Gli hooks "AtExit" vengono registrati utilizzando l'API `node::AtExit`.

#### void AtExit(callback, args)

* `callback` {void (*)(void*)} Un puntatore alla funzione per una "call at exit".
* `args` {void\*} Un puntatore per passare ad un "callback at exit".

Registra gli "exit hooks" che vengono eseguiti dopo che il ciclo di eventi è terminato ma prima che la VM venga distrutta.

AtExit accetta due parametri: un puntatore ad una funzione di callback da eseguire all'uscita (at exit), ed un puntatore a dati contestuali untyped da passare a tale callback.

I callback vengono eseguiti nell'ordine last-in first-out (ultimo ad entrare, primo ad uscire).

Il seguente `addon.cc` implementa AtExit:

```cpp
// addon.cc
#include <assert.h>
#include <stdlib.h>
#include <node.h>

namespace demo {

using node::AtExit;
using v8::HandleScope;
using v8::Isolate;
using v8::Local;
using v8::Object;

static char cookie[] = "yum yum";
static int at_exit_cb1_called = 0;
static int at_exit_cb2_called = 0;

static void at_exit_cb1(void* arg) {
  Isolate* isolate = static_cast<Isolate*>(arg);
  HandleScope scope(isolate);
  Local<Object> obj = Object::New(isolate);
  assert(!obj.IsEmpty()); // Afferma che la VM è ancora funzionante
  assert(obj->IsObject());
  at_exit_cb1_called++;
}

static void at_exit_cb2(void* arg) {
  assert(arg == static_cast<void*>(cookie));
  at_exit_cb2_called++;
}

static void sanity_check(void*) {
  assert(at_exit_cb1_called == 1);
  assert(at_exit_cb2_called == 2);
}

void init(Local<Object> exports) {
  AtExit(at_exit_cb2, cookie);
  AtExit(at_exit_cb2, cookie);
  AtExit(at_exit_cb1, exports->GetIsolate());
  AtExit(sanity_check);
}

NODE_MODULE(addon, init)

}  // namespace demo
```

Testa in JavaScript eseguendo:

```js
// test.js
require('./build/Release/addon');
```