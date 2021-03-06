[![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/cstrahan/capnp-ruby/trend.png)](https://bitdeli.com/free "Bitdeli Badge")

![Cap'n Proto][logo]

# Ruby Edition

This here is a [Ruby][ruby] wrapper for the official C++ implementation of [Cap'n Proto][capnp].

[![Build Status][travis-badge]][travis-link]

# Installing

First [install libcapnp][libcapnp-install], then install the gem:

```bash
gem install capn_proto-rpc --pre
```

or add this to your Gemfile


```bash
gem capn_proto-rpc
```

The native extension for this gem requires a C++ compiler with C++11 features.
I've hardcoded compiler flags directly on the makefile in order to make the install easier.

# RPC Client Example    
note: the schema file, client example and the server example can be found in lib/tests as a minitest.   

The following examples uses this schema:

```CapnProto
# file hidraCordatus.capnp

struct Task {
  dataint @0 :Int32;
  madeBy @1 :Text;
}

interface Employer {
  getWorker @0 () -> ( worker :Worker );
}

interface Worker {
  put23 @0 (taskToProcess :Task) -> (taskProcessed :Task);
}

```
Load all the schemas and methods that will be used then create an EzRpcClient and from it get our client.
```ruby
require 'capn_proto'

module Hydra extend CapnProto::SchemaLoader
  load_schema('./tests/hidraCordatus.capnp')
end

employer_schema =   Hydra::Employer.schema
get_worker_method = Hydra::Employer.method! 'getWorker'
put23method =       Hydra::Worker.method!   'put23'

ezclient = CapnProto::EzRpcClient.new("127.0.0.1:1337",employer_schema)
client = ezclient.client

```
Create a request of the method "getWorker" who is in the variable get_worker_method above.
Then, send it storing the pipelined request.
```ruby
request = client.request(get_worker_method)
pipelinedRequest = request.send
```
get the returned "worker" set the method that we want to request on it and then set
the parameters to be requested, in this case we set dataint to 0.

```ruby
pipelinedRequest.get('worker').method = put23method
pipelinedRequest.taskToProcess.dataint(0)
```
now we wait for the results (note that this is the only line that blocks). while we are waiting
the Global interpreter lock is released so we can run ruby code on other threads.
Also note that we use ezclient as a waitscope.
```ruby
results = pipelinedRequest.send.wait(ezclient)
puts results.taskProcessed.dataint
puts results.taskProcessed.madeBy
```

# RPC server Example
```ruby
require 'capn_proto'

module Hydra extend CapnProto::SchemaLoader
  load_schema('./tests/hidraCordatus.capnp')
end

class WorkerServer < CapnProto::CapabilityServer
  def initialize(i)
    @madeBy = "made by worker ##{i}"
    super(Hydra::Worker.schema)
  end

  def put23(context)
    n = context.getParams.taskToProcess.dataint
    context.getResults.taskProcessed.dataint = n + 23
    context.getResults.taskProcessed.madeBy = @madeBy
  end
end

class EmployerServer < CapnProto::CapabilityServer
  def initialize(wp)
    @worker_pool = wp
    @currentWorker = 0
    super(Hydra::Employer.schema)
  end

  def get_a_Worker
    @currentWorker += 1
    @worker_pool[@currentWorker % @worker_pool.size]
  end

  def getWorker(context)
    context.getResults.worker = get_a_Worker
  end
end

```
note that the name of the methods is exactly the same as the name of the function that is defined on the schema and recieves only one argument. This argument is a callContext, you can use the method **getParams** to get the parameters passed  to the called method or
use **getResults** to set the results of the request.   
regarding to the example, EmployerServer will serve WorkerServers to the clients.

```ruby
workers = []
10.times do |i|
  workers << WorkerServer.new(i)
end


e = CapnProto::EzRpcServer.new(EmployerServer.new(workers), "*:1337")
puts "serving EmployerServer on 1337..."
e.run

```
create ten workers, then a EzRpcServer wich binds to port 1337. Then run it.
```
the results of running the server/client pair is :

23
"made by worker #1"
23
"made by worker #2"
23
"made by worker #3"
23
...
```
# Structs Example

```ruby
require 'capn_proto'

module AddressBook extend CapnProto::SchemaLoader
  load_schema("addressbook.capnp")
end

def write_address_book(file)
  addresses = AddressBook::AddressBook.new_message
  people = addresses.initPeople(2)

  alice = people[0]
  alice.id = 123
  alice.name = 'Alice'
  alice.email = 'alice@example.com'
  alice_phones = alice.initPhones(1)
  alice_phones[0].number = "555-1212"
  alice_phones[0].type = 'mobile'
  alice.employment.school = "MIT"

  bob = people[1]
  bob.id = 456
  bob.name = 'Bob'
  bob.email = 'bob@example.com'
  bob_phones = bob.initPhones(2)
  bob_phones[0].number = "555-4567"
  bob_phones[0].type = 'home'
  bob_phones[1].number = "555-7654"
  bob_phones[1].type = 'work'
  bob.employment.unemployed = nil

  addresses.write(file)
end

def print_address_book(file)
  addresses = AddressBook::AddressBook.read_from(file)

  addresses.people.each do |person|
    puts "#{person.name} : #{person.email}"

    person.phones.each do |phone|
      puts "#{phone.type} : #{phone.number}"
    end

    if person.employment.unemployed?
      puts "unemployed"
    if person.employment.employer?
      puts "employer: #{person.employment.employer}"
    if person.employment.school?
      puts "student at: #{person.employment.school}"
    if person.employment.selfEmployed?
      puts "self employed"
    end
  end
end

if __FILE__ == $0
  file = File.open("addressbook.bin", "wb")
  write_address_book(file)

  file = File.open("addressbook.bin", "rb")
  print_address_book(file)
end
```

# Status

What's implemented:
- Schema parsing/loading
- Message reading
  - From byte string
  - From file descriptor
- Message writing
  - To byte string
  - To file descriptor
- RPC
  - loading InterfaceSchema and their methods
  - RPC client
  - RPC server

What's to come:
- More reading/writing mechanisms:
  - Packing/unpacking
- Extensive test coverage
- Proper support for [JRuby][jruby]
- There is a known bug where the servers don't exits when pressing control-c. It only exits after
pressing control-c and then a request is made from a client.

[logo]: https://raw.github.com/cstrahan/capnp-ruby/master/media/captain_proto_small.png "Cap'n Proto"
[ruby]: http://www.ruby-lang.org/ "Ruby"
[capnp]: http://kentonv.github.io/capnproto/ "Cap'n Proto"
[jruby]: http://jruby.org/ "JRuby"
[libcapnp-install]: http://kentonv.github.io/capnproto/install.html "Installing Cap'n Proto"
[mit-license]: http://opensource.org/licenses/MIT "MIT License"

[travis-link]: https://travis-ci.org/cstrahan/capnp-ruby
[travis-badge]: https://travis-ci.org/cstrahan/capnp-ruby.png?branch=master
