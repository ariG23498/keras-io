# Models API

There are three ways to create Keras models:

- The [Sequential model](/keras_core/guides/sequential_model), which is very straightforward (a simple list of layers),
    but is limited to single-input, single-output stacks of layers (as the name gives away).
- The [Functional API](/keras_core/guides/functional_api), which is an easy-to-use, fully-featured API that supports arbitrary model architectures.
    For most people and most use cases, this is what you should be using. This is the Keras "industry strength" model.
- [Model subclassing](/keras_core/guides/making_new_layers_and_models_via_subclassing), where you implement everything from scratch on your own.
    Use this if you have complex, unconventional research use cases.


## Models API overview

{{toc}}

