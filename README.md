# The Neural Turing Machine

This code tries to implement the Neural Turing Machine, as found in 
https://arxiv.org/abs/1410.5401, as a backend neutral recurrent keras layer.

NOTE:
* There is a nicely formatted paper describing the rough idea of the NTM, implementation difficulties and which discusses the
  copy experiment. It is available here in the repository as The_NTM_-_Introduction_And_Implementation.pdf. 
* You may want to change the LOGDIR_BASE in testing_utils.py to something that works for you or just set a symbolic
  link.
* So far one read and one write head is hardcoded into the system. This will probably be fixed in a future update.


### User guide
For a quick start on the copy Task, type 

    python main.py -v ntm

while in a python enviroment which has tensorflow, keras, numpy. tensorflow-gpu is recommend, as everything is about 20x
faster. In my case this takes about 100 minutes on a NVIDIA GTX 1050 Ti.
The -v is optional and offers much more detailed information about the achieved accuracy, and also after every training
epoch.

Compare that with

    python main.py lstm

This builds 3 layers of LSTM with and goes through the same testing procedure
as above, which for me resulted in a training time of approximately 1h (same GPU) and 
(roughly) 100%, 100%, 94%, 50%, 50% accuracy at the respective test lengths. 

Have fun with the results!


### API
From the outside, this implementation looks like a regular recurrent layer in keras.
It has however a number of non-obvious parameters:

*  n_width: This is the width of the memory matrix. Increasing this increases computational complexity in O(n). The
   controller shape is not dependant on this, making weight transfer possible.

*  m_depth: This is the depth of the memory matrix. Increasing this increases the number of trainable weights in O(n^2). It also changes controller shape. 

*  controller_model: This parameter allows you to place a keras model of appropriate shape as the controller. The
appropriate shape can be calculated via controller_input_output_shape. If None is set, a single dense layer will be
used. 

More or less minimal code example:

    from keras.models import Sequential
    from keras.optimizers import Adam
    from ntm import NeuralTuringMachine as NTM

    model = Sequential()
    model.name = "NTM_-_" + controller_model.name

    ntm = NTM(output_dim, n_slots=50, m_depth=20, shift_range=3,
              controller_model=None,
              return_sequences=True,
              input_shape=(None, input_dim), 
              batch_size = 100)
    model.add(ntm)

    sgd = Adam(lr=learning_rate, clipnorm=clipnorm)
    model.compile(loss='binary_crossentropy', optimizer=sgd, metrics = ['binary_accuracy'], sample_weight_mode="temporal")

What if we instead want a more complex controller? Design it, e.g. double LSTM:

    controller = Sequential()
    controller.name=ntm_controller_architecture
    controller.add(LSTM(units=150,
                                stateful=True,
                                implementation=2,   # best for gpu. other ones also might not work.
                                batch_input_shape=(batch_size, None, controller_input_dim)))
    controller.add(LSTM(units=controller_output_dim,
                                activation='sigmoid',
                                stateful=True,
                                implementation=2))   # best for gpu. other ones also might not work.

    controller.compile(loss='binary_crossentropy', optimizer=sgd, metrics = ['binary_accuracy'], sample_weight_mode="temporal")

And now use the same code as above, only with controller_model=controller.

Note that we used sigmoid as the last activation layer! This is currently necessary. Or at least another activation
layer with idles at 0.5 and lies in the range of (0,1).

Note that a correct controller_input_dim and controller_output_dim can be calculated via controller_input_output_shape:

    from ntm import controller_input_output_shape
    controller_input_dim, controller_output_dim = ntm.controller_input_output_shape(
                input_dim, output_dim, m_depth, n_slots, shift_range, 1,1) 


Also note that every statefull controller must carry around his own state, as was done here with 

    stateful=True





## TODO:
* Arbitrary number of read and write heads
* Support of masking, and maybe dropout, one has to reason about it theoretically first.
* support for get and set config to better enable model saving
* A bit of code cleaning: especially the controller output splitting is ugly as hell.
* Although it should be backend neutral, some testing with other backends might be nice.
* Support for arbitrary activation functions would be nice, currently restricted to sigmoid.
