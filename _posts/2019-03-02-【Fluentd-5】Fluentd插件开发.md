---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Fluentd
---

# 一. 项目结构
Fluentd PluginGem项目结构可以使用`fluent-plugin-generate`工具一键生成，如下所示

```
➜  fluent-plugin-generate input http2
License: Apache-2.0
	create Gemfile
	create README.md
	create Rakefile
	create fluent-plugin-http2.gemspec
	create lib/fluent/plugin/in_http2.rb
	create test/helper.rb
	create test/plugin/test_in_http2.rb
Initialized empty Git repository in /Users/zj-db0972/Downloads/tm/fluent-plugin-http2/.git/
➜  ls
fluent-plugin-http2
➜  tree
.
└── fluent-plugin-http2
    ├── Gemfile                       //定义你的应用依赖哪些第三方包，bundle根据该配置去寻找这些包
    ├── LICENSE
    ├── README.md
    ├── Rakefile                      //定义要执行的Rake的命令
    ├── fluent-plugin-http2.gemspec   //Gem包描述
    ├── lib
    │   └── fluent
    │       └── plugin
    │           └── in_http2.rb       //in_http2插件
    └── test
        ├── helper.rb
        └── plugin
            └── test_in_http2.rb      //in_http2插件单元测试

6 directories, 8 files
```

# 二. 插件开发
1. 开发Fluentd Plugin之前，我们先来了解Plugin的生命周期，一个Plugin从初始化到启动到结束会经历以下几个过程  
   `initialize -> configure -> start -> process events -> shutdown`
2. 开发Fluentd Plugin需要各自继承Fluent模块下的基类，如有需要并实现以下几个函数，每个方法必须调用`super`
    * def initialize: 实例初始化函数 (如果没有需要初始化可以不用实现)
    * def configure(conf): configure函数用于读取配置，会在start函数之前调用  (如果没有需要初始化可以不用实现)
    * def start: start函数用于启动plugin实例  (如果没有需要初始化可以不用实现)
    * def shutdown: shutdown函数用于停止plugin实例  (如果没有需要初始化可以不用实现)
3. 开发Ruby推荐使用[RubyMine](https://www.jetbrains.com/ruby/)，本文所有代码基于`Fluentd v0.12`版本 和 `Ruby 2.3.8`版本开发，并经过单元测试验证通过。(Fluentd v0.12和v1.x两个版本之间存在差异，本文所有代码已经配置都是基于v0.12，如果在v1.0上代码需要做变更调整)
4. 使用`RVM`来管理Ruby版本，使用`Gem`安装指定的依赖包
    * 安装RVM
        * `echo "gem: --no-document" >> ~/.gemrc`
        * `curl -L https://get.rvm.io | bash -s stable --auto-dotfiles --autolibs=enable --rails`
    * 安装Ruby 2.3.8版本
        * `rvm install 2.3.8`
        * `rvm use ruby-2.3.8`
    * RubyMine设置Ruby SDK
        * `Perferences -> Languages * Frameworks -> Ruby SDK and Gems 选择Ruby 版本`
    * 安装Fluentd和相关依赖
        * `gem install fluentd -v "~> 0.12.0" --no-ri --no-rdoc`
        * `gem install fluent-mixin-config-placeholders`
        * `gem install fluent-plugin-prometheus -v "~> 0.4.0"`
        * `gem install rr`
        * `gem install test-unit-rr`


## ① Input Plugins
开发Input Plugin需要遵循以下几点原则

1. 自定义类需要继承`Fluent::Plugin::Input`基类
2. 注册插件名称，插件名称可以在fluentd.conf配置文件里面`source`命令配置`@type`参数使用

很多情况下 Input插件都是通过`start`方法开启一个**timer(定时器)** 或 **开启一个thread(线程)** 或 **开启一个server(网络服务)** 监听某个端口，然后通过`router.emit(tag, time, record)`提交一个日志事件到Fluentd的路由引擎进行处理。

```
#
# Copyright 2019- 玄苦
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
require 'fluent/input'

 module Fluent
    class SampleInput < Fluent::Input
      attr_reader :in_records, :in_bytes

      # First, register the plugin. NAME is the name of this plugin
      # and identifies the plugin in the configuration file.
      Fluent::Plugin.register_input('in_sample', self)

      # config_param defines a parameter. You can refer a parameter via @port instance variable
      # :default means this parameter is optional
      config_param :port, :integer, default: 8888

      # initialize
      def initialize
        super

        # 类变量定义
        @in_records = 0
        @in_bytes = 0.0
      end

      # This method is called before starting.
      # 'conf' is a Hash that includes configuration parameters.
      # If the configuration is invalid, raise Fluent::ConfigError.
      def configure(conf)
        super

        # You can also refer to raw parameter via conf[name].
        @port = conf['port']
      end

      # This method is called when starting.
      # Open sockets or files and create a thread here.
      def start
        super
        # my own start-up code
        @watcher = Thread.new(&method(:run))
      end

      # This method is called when shutting down.
      def shutdown
        # my own shutdown code
        @watcher.terminate
        @watcher.join
        super
      end

      # This method is called by thread.
      def run
        while true
          tag = "in_sample_tag"
          time = Fluent::Engine.now
          record = {"message"=>"hello my name is xuanku ~"}
          # emit 2 fluentd router engine
          router.emit(tag, time, record)

          @in_records += 1
          @in_bytes += record.to_msgpack.bytesize

          # wait next record
          sleep 1.0
        end
      end

    end
  end
```

单元测试
```
# test/plugin/test_in_your_own.rb
require 'test/unit'
require 'fluent/test'
require 'fluent/test/helpers'

# your own plugin
require_relative '../../lib/fluent/plugin/in_sample'

class SampleInputTest < Test::Unit::TestCase
  def setup
    # this is required to setup router and others
    Fluent::Test.setup
  end

  # 配置
  CONFIG = %[
    port 8080
  ]

  # default configuration for tests
  def create_driver(conf = CONFIG)
    Fluent::Test::InputTestDriver.new(Fluent::SampleInput).configure(conf)
  end

  sub_test_case 'test SampleInput plugin' do
    test 'test SampleInput plugin' do
      d = create_driver
      d.run

      # assert
      assert_equal("8080", d.instance.instance_variable_get("@port"))
      assert_equal(1, d.instance.in_records)
      assert_equal(35.0, d.instance.in_bytes)

      d.expected_emits.each do |tag, time, record|
        assert_equal("in_sample_tag", tag)
        assert_equal({"message"=>"hello my name is xuanku ~"}, record)
        assert(time.is_a?(Fluent::EventTime))
      end
    end
  end

end
```

## ② Output Plugin
Output Plugin有三种类型，Buffered Output Plugins、Time Sliced Output Plugins 和 Non-buffered Output Plugins。

1. Buffered Output Plugins: 带有buffer的Output插件
    * 必须继承自Fluent::BufferedOutput这个基类
    * 必须实现def write(chunk)方法，用于写chunk到具体的存储媒介
   
```
#
# Copyright 2019- 玄苦
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

require 'fluent/output'

module Fluent
   class SampleBufferedOutput < Fluent::BufferedOutput
    attr_reader :out_bytes

    # First, register the plugin. NAME is the name of this plugin
    # and identifies the plugin in the configuration file.
    Fluent::Plugin.register_output('out_buffered_sample', self)

    # config_param defines a parameter. You can refer a parameter via @path instance variable
    # Without :default, a parameter is required.
    config_param :path, :string

    # initialize
    def initialize
      super

      # 类变量定义
      @out_bytes = 0.0
    end

    # This method is called before starting.
    # 'conf' is a Hash that includes configuration parameters.
    # If the configuration is invalid, raise Fluent::ConfigError.
    def configure(conf)
      super

      # You can also refer raw parameter via conf[name].
      @path = conf['path']
    end

    # This method is called when starting.
    # Open sockets or files here.
    def start
      super
    end

    # This method is called when shutting down.
    # Shutdown the thread and close sockets or files here.
    def shutdown
      super
    end

    # This method is called when an event reaches to Fluentd.
    # Convert the event to a raw string.
    def format(tag, time, record)
      [tag, time, record].to_json + "\n"
    end

    # This method is called every flush interval. Write the buffer chunk
    # to files or databases here.
    # 'chunk' is a buffer chunk that includes multiple formatted
    # events. You can use 'data = chunk.read' to get all events and
    # 'chunk.open {|io| ... }' to get IO objects.
    #
    # NOTE! This method is called by internal thread, not Fluentd's main thread. So IO wait doesn't affect other plugins.
    def write(chunk)
      @out_bytes += chunk.read.to_s.bytesize
    end

  end
end
```

单元测试
```
# test/plugin/test_in_your_own.rb
require 'fluent/test'
require 'fluent/test/helpers'

# your own plugin
require_relative '../../lib/fluent/plugin/out_buffered_sample'

class SampleBufferedOutputTest < Test::Unit::TestCase
    # this is required to setup router and others
    def setup
      Fluent::Test.setup
    end

    CONFIG = %[
      buffer_type file
      buffer_path /Users/zj-db0972/Downloads/tm/fluent-plugin-develop/buffer/sample_buffer_out
      buffer_chunk_limit 64M
      buffer_queue_limit 512
      flush_interval 3s

      path /var/log/system.log
    ]
    # default configuration for tests
    def create_driver(conf = CONFIG)
      Fluent::Test::BufferedOutputTestDriver.new(Fluent::SampleBufferedOutput).configure(conf)
    end

    sub_test_case 'tests for write' do
      test 'to test write' do
        d = create_driver
        d.instance.start

        tag = "out_buffered_sample_tag"
        time = Fluent::Engine.now
        record = {"message" => "hello, my name is xunku ~"}
        es = Fluent::OneEventStream.new(time, record)

        # run
        d.run do
            d.instance.emit(tag, es, Fluent::NullOutputChain.instance)
            d.instance.force_flush
        end

        # assert
        assert_equal"file", d.instance.instance_variable_get("@buffer_type")
        assert_equal("/var/log/system.log", d.instance.instance_variable_get("@path"))
        assert_equal(67108864, d.instance.instance_variable_get("@buffer").instance_variable_get("@buffer_chunk_limit"))
        assert_equal(512, d.instance.instance_variable_get("@buffer").instance_variable_get("@buffer_queue_limit"))
        assert_equal(3, d.instance.instance_variable_get("@flush_interval"))
        assert_equal(79.0, d.instance.out_bytes)
      end
    end

  end
```

2. Time Sliced Output Plugins: 时间切片的输出插件
    * 必须继承自Fluent::BufferedOutput这个基类
    * 必须实现def write(chunk)方法，用于写chunk到具体的存储媒介
    
```
#
# Copyright 2019- 玄苦
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

require 'fluent/output'

module Fluent
   class SampleTimeSlicedOutput < Fluent::TimeSlicedOutput
    attr_reader :out_bytes

    # First, register the plugin. NAME is the name of this plugin
    # and identifies the plugin in the configuration file.
    Fluent::Plugin.register_output('out_time_sliced_sample', self)

    # config_param defines a parameter. You can refer a parameter via @path instance variable
    # Without :default, a parameter is required.
    config_param :path, :string

    # initialize
    def initialize
      super

      # 类变量定义
      @out_bytes = 0.0
    end

    # This method is called before starting.
    # 'conf' is a Hash that includes configuration parameters.
    # If the configuration is invalid, raise Fluent::ConfigError.
    def configure(conf)
      super

      # You can also refer raw parameter via conf[name].
      @path = conf['path']
    end

    # This method is called when starting.
    # Open sockets or files here.
    def start
      super
    end

    # This method is called when shutting down.
    # Shutdown the thread and close sockets or files here.
    def shutdown
      super
    end

    # This method is called when an event reaches to Fluentd.
    # Convert the event to a raw string.
    def format(tag, time, record)
      [tag, time, record].to_json + "\n"
    end

    # You can use 'chunk.key' to get sliced time. The format of 'chunk.key'
    # can be configured by the 'time_format' option. The default format is %Y%m%d.
    def write(chunk)
      @out_bytes += chunk.read.to_s.bytesize
    end

  end
end
```

单元测试
```
# test/plugin/test_in_your_own.rb
require 'fluent/test'
require 'fluent/test/helpers'

# your own plugin
require_relative '../../lib/fluent/plugin/out_time_sliced_sample'

class SampleTimeSlicedOutputTest < Test::Unit::TestCase
    # this is required to setup router and others
    def setup
      Fluent::Test.setup
    end

    CONFIG = %[
      buffer_path /Users/zj-db0972/Downloads/tm/fluent-plugin-develop/buffer/sample_buffer_out
      buffer_chunk_limit 64M
      buffer_queue_limit 512
      flush_interval 3s

      path /var/log/system.log
    ]
    # default configuration for tests
    def create_driver(conf = CONFIG)
      Fluent::Test::OutputTestDriver.new(Fluent::SampleTimeSlicedOutput).configure(conf)
    end

    sub_test_case 'tests for write' do
      test 'to test write' do
        d = create_driver
        d.instance.start
        tag = "out_buffered_sample_tag"
        time = Fluent::Engine.now
        record = {"message" => "hello, my name is xunku ~"}
        es = Fluent::OneEventStream.new(time, record)

        # run
        d.run do
          d.instance.emit(tag, es, Fluent::NullOutputChain.instance)
          d.instance.force_flush
        end

        # assert
        assert_equal"file", d.instance.instance_variable_get("@buffer_type")
        assert_equal("/var/log/system.log", d.instance.instance_variable_get("@path"))
        assert_equal(67108864, d.instance.instance_variable_get("@buffer").instance_variable_get("@buffer_chunk_limit"))
        assert_equal(512, d.instance.instance_variable_get("@buffer").instance_variable_get("@buffer_queue_limit"))
        assert_equal(3, d.instance.instance_variable_get("@flush_interval"))
        assert_equal(79.0, d.instance.out_bytes)
      end
    end

  end
```

3. Non-buffered Output Plugins: 没有buffer的插件
    * 必须继承自Fluent::Output这个基类
    * 必须实现def emit(tag, es, chain)方法，用于提交日志事件到Fluentd路由引擎

```
#
# Copyright 2019- 玄苦
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

require 'fluent/output'

module Fluent
   class SampleOutput < Fluent::Output
    attr_reader :out_records, :out_bytes

    # First, register the plugin. NAME is the name of this plugin
    # and identifies the plugin in the configuration file.
    Fluent::Plugin.register_output('out_sample', self)

    # config_param defines a parameter. You can refer a parameter via @path instance variable
    # Without :default, a parameter is required.
    config_param :path, :string

    # initialize
    def initialize
      super

      # 类变量定义
      @out_records = 0
      @out_bytes = 0.0
    end

    # This method is called before starting.
    # 'conf' is a Hash that includes configuration parameters.
    # If the configuration is invalid, raise Fluent::ConfigError.
    def configure(conf)
      super

      # You can also refer raw parameter via conf[name].
      @path = conf['path']
    end

    # This method is called when starting.
    # Open sockets or files here.
    def start
      super
    end

    # This method is called when shutting down.
    # Shutdown the thread and close sockets or files here.
    def shutdown
      super
    end

    # This method is called when an event reaches Fluentd.
    # 'es' is a Fluent::EventStream object that includes multiple events.
    # You can use 'es.each {|time,record| ... }' to retrieve events.
    # 'chain' is an object that manages transactions. Call 'chain.next' at
    # appropriate points and rollback if it raises an exception.
    #
    # NOTE! This method is called by Fluentd's main thread so you should not write slow routine here. It causes Fluentd's performance degression.
    def emit(tag, es, chain)
      chain.next
      es.each {|time,record|
        record.store("amount", 250.250)
        # add couter and bytes
        @out_records += 1
        @out_bytes += record.to_msgpack.bytesize
      }
    end

  end
end
```

单元测试
```
# test/plugin/test_in_your_own.rb
require 'fluent/test'
require 'fluent/test/helpers'

# your own plugin
require_relative '../../lib/fluent/plugin/out_sample'

class SampleOutputTest < Test::Unit::TestCase
    # this is required to setup router and others
    def setup
      Fluent::Test.setup
    end

    # 配置
    CONFIG = %[
      path /var/log/system.log
    ]
    # default configuration for tests
    def create_driver(conf = CONFIG)
      Fluent::Test::OutputTestDriver.new(Fluent::SampleOutput).configure(conf)
    end

    sub_test_case 'tests for emit' do
      test 'to test emit' do
        d = create_driver
        tag = "out_sample_tag"
        time = Fluent::Engine.now
        record = {"message" => "hello, my name is xunku ~"}
        es = Fluent::OneEventStream.new(time, record)

        # run
        d.run do
          d.instance.emit(tag, es, Fluent::NullOutputChain.instance)
        end

        # assert
        assert_equal(1, d.instance.out_records)
        assert_equal(51.0, d.instance.out_bytes)
      end
    end

  end
```

## ③ Filter Plugin
开发Filter Plugin需要遵循以下几点原则

1. 自定义类需要继承`Fluent::Filter`这个基类
2. 注册插件名称，插件名称可以直接在fluentd.conf配置文件里面`filter`命令配置`@type`参数使用
3. 必须实现def filter(tag, time, record)方法，用于处理日志record
    
```
#
# Copyright 2019- 玄苦
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

require 'fluent/filter'

module Fluent
   class FilterSample < Fluent::Filter
    attr_reader :filter_records, :filter_bytes

    # First, register the plugin. NAME is the name of this plugin
    # and identifies the plugin in the configuration file.
    Fluent::Plugin.register_output('filter_sample', self)

    # initialize
    def initialize
      super

      # 类变量定义
      @filter_records = 0
      @filter_bytes = 0.0
    end

    # This method is called before starting.
    # 'conf' is a Hash that includes configuration parameters.
    # If the configuration is invalid, raise Fluent::ConfigError.
    def configure(conf)
      super
    end

    # This method is called when starting.
    # Open sockets or files here.
    def start
      super
    end

    # This method is called when shutting down.
    # Shutdown the thread and close sockets or files here.
    def shutdown
      super
    end

    # This method implements the filtering logic for individual filters
    # It is internal to this class and called by filter_stream unless
    # the user overrides filter_stream.
    #
    # Since our example is a pass-thru filter, it does nothing and just
    # returns the record as-is.
    # If returns nil, that records are ignored.
    def filter(tag, time, record)
      # TODO 业务逻辑
      begin
        record.store("message","rewrite record message")
        # 统计
        @filter_records += 1
        @filter_bytes += record.to_msgpack.bytesize
      rescue => e
        log.warn "failed to filter events", error: e
        log.warn_backtrace
      end

      record
    end

  end
end
```

单元测试
```
# test/plugin/test_in_your_own.rb
require 'fluent/test'
require 'fluent/test/helpers'

# your own plugin
require_relative '../../lib/fluent/plugin/filter_sample'

class SampleFilterTest < Test::Unit::TestCase
    # this is required to setup router and others
    def setup
      Fluent::Test.setup
    end

    # 配置
    CONFIG = %[
    ]
    # default configuration for tests
    def create_driver(conf = CONFIG)
      Fluent::Test::FilterTestDriver.new(Fluent::FilterSample).configure(conf)
    end

    sub_test_case 'tests for filter' do
      test 'to test filter' do
        d = create_driver
        tag = "filter_sample_tag"
        time = Fluent::Engine.now
        record = {"message" => "hello, my name is xunku ~"}

        # run
        d.run do
          d.instance.filter(tag, time, record)
        end

        # assert
        assert_equal(1, d.instance.filter_records)
        assert_equal(32.0, d.instance.filter_bytes)
      end
    end

  end
```

## ④ Parser Plugin
开发Parser Plugin需要遵循以下几点原则

1. 自定义类需要继承`Fluent::Parser`这个基类
2. 注册插件名称，插件名称可以在fluentd.conf配置文件`source`里面配置`format`参数使用
3. 必须实现def parse(text)方法，用于解析文本日志

```
#
# Copyright 2019- 玄苦
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

require 'fluent/parser'

module Fluent
  class ParserSample < Fluent::Parser
    # Register this parser as "time_key_value"
    Fluent::Plugin.register_parser("parser_sample", self)

    # 插件参数
    config_param :delimiter, :string, default: " "   # delimiter is configurable with " " as default
    config_param :time_format, :string, default: nil # time_format is configurable

    # This method is called before starting.
    # 'conf' is a Hash that includes configuration parameters.
    # If the configuration is invalid, raise Fluent::ConfigError.
    def configure(conf)
      super

      if @delimiter.length != 1
        raise ConfigError, "delimiter must be a single character. #{@delimiter} is not."
      end

      # TimeParser class is already given. It takes a single argument as the time format
      # to parse the time string with.
      @time_parser = Fluent::TextParser::TimeParser.new(@time_format)
    end

    # parse function
    def parse(text)
      # TODO 业务逻辑
      time, key_values = text.split(" ", 2)
      time = @time_parser.parse(time)
      record = {}
      key_values.split(@delimiter).each { |kv|
        k, v = kv.split("=", 2)
        record[k] = v
      }
      yield time, record
    end
  end
end
```

单元测试
```
# test/plugin/test_in_your_own.rb
require 'fluent/test'
require 'fluent/test/helpers'

# your own plugin
require_relative '../../lib/fluent/plugin/parser_sample'

class ParserSampleTest < Test::Unit::TestCase
    # this is required to setup router and others
    def setup
      Fluent::Test.setup
    end

    # 配置
    CONFIG = %[
      format parser_sample
      delimiter ;
      time_format %Y-%m-%dT%H:%M:%S
    ]

    # default configuration for tests
    def create_driver(conf = CONFIG)
      Fluent::Test::ParserTestDriver.new(Fluent::ParserSample).configure(conf)
    end

    # test case
    sub_test_case 'test parser plugin' do
      test 'test parser plugin' do
        d = create_driver
        text = '2019-03-01T00:00:00 name=cgl;age=100;action=debugging'

        expected_time = 1551369600
        expected_record = {
            "name" => "cgl",
            "age" => "100",
            "action" => "debugging",
        }

        # assert
        d.instance.parse(text) do |time, record|
          assert_equal(expected_time, time)
          assert_equal(expected_record, record)
        end
      end
    end

  end
```

## ⑤ Formatter Plugin
开发Formatter Plugin需要遵循以下几点原则

1. 自定义类需要继承`Fluent::Formatter`这个基类
2. 注册插件名称，插件名称可以直接在fluentd.conf配置文件里面`match`命令配置`format`参数使用
3. 必须实现def format(tag, time, record)方法用于格式化日志record

```
#
# Copyright 2019- 玄苦
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

require 'fluent/formatter'

module Fluent
  class FormatterSample < Fluent::Formatter
    Fluent::Plugin.register_parser("formatter_sample", self)

    # 插件参数
    config_param :csv_fields, :array, value_type: :string

    # This method does further processing. Configuration parameters can be
    # accessed either via "conf" hash or member variables.
    def configure(conf)
      super
    end

    # This is the method that formats the data output.
    def format(tag, time, record)
      values = []

      # Look up each required field and collect them from the record
      @csv_fields.each do |field|
        v = record[field]
        if not v
          $log.error "#{field} is missing."
        end
        values << v
      end

      # Output by joining the fields with a comma
      values.join(",")
    end
  end
end
```

单元测试
```
# test/plugin/test_in_your_own.rb
require 'fluent/test'
require 'fluent/test/helpers'

# your own plugin
require_relative '../../lib/fluent/plugin/formatter_sample'

class FormatterSampleTest < Test::Unit::TestCase
    # this is required to setup router and others
    def setup
      Fluent::Test.setup
    end

    # 配置
    CONFIG = %[
      csv_fields name,age,addr
    ]

    # default configuration for tests
    def create_driver(conf = CONFIG)
      Fluent::Test::FormatterTestDriver.new(Fluent::FormatterSample).configure(conf)
    end

    # test case
    sub_test_case 'test format record' do
      test 'test format record ' do
        d = create_driver
        tag = "test"
        time = Fluent::Engine.now
        record = {
            "message" => "This is message",
            "name" => "cgl",
            "age" => 100,
            "addr" => "xiamen",
        }

        # format
        formatted = d.instance.format(tag, time, record)

        # assert
        assert_equal("cgl,100,xiamen", formatted)
      end
    end

  end
```

# 三. 插件管理
当我们写完插件后或者要添加第三方插件的时候可以使用以下2个方式

1. 使用gem安装  
   $ gem install fluent-plugin-grep
2. 添加插件到/etc/fluent/plugin目录  
   1). /etc/fluent/plugin是Fluentd默认加载的路径，只要把plugin添加到这个目录Fluentd就会自动加载  
   2). 例如有这个插件/etc/fluent/plugin/out_foo.rb, 我们就可以在match命令直接使用`@type foo`


