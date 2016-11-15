---
layout: post
title: 修改 Cucumber HTML 报告
date: 2016-04-13 09:45
disqus: y
---

后台服务是 JSON-RPC 风格的，所以 Scenario 都是这样的  
  
```gherkin
Scenario: login successful
  When I set request body from "features/examples/login.json”
  When I send a POST request to "/app_api/session/authenticate/“
  Then the response status should be "200"
  Then the JSON response should follow "features/schemas/login_response.schema.json"
  Then the JSON response should have key "$.result.username" with "cucumbertest”
```
  
HTML 报告是这样的  
![](/images/2016/b3762a5324b8b6d6b9d4756040ce320d.png)  

项目目录结构是这样的  
  
```
.
└── features
    ├── examples
    │   └── login.json
    ├── schemas
    │   └── login_response.schema.json
    ├── step_definitions
    │   └── setp.rb
    ├── support
    │   └── env.rb
    └── user.feature
```
  
现在想把 features/examples/login.json 和 features/schemas/login_response.schema.json 这部分加上超链接，点击后显示相应文件内容，把  
`<span class="param">features/examples/login.json</span>`  
改为  
`<a href="features/examples/login.json" target="_blank">features/examples/login.json</a>`  
  
在[源码里搜索`span class="param”`](https://github.com/cucumber/cucumber-ruby/search?utf8=%E2%9C%93&q=span+class=%22param%22&type=Code)，发现拼接这部分 HTML 的是 lib/cucumber/formatter/html.rb 文件里的 build_step 方法  
  
在 features/support 目录下新建 html_with_link.rb 文件，输入  
  
```ruby
require 'cucumber/formatter/html'

module Cucumber
  module Formatter
    class HtmlWithLink < Html
      def build_step(keyword, step_match, status)
        # step_name = step_match.format_args(lambda{|param| %{<span class="param">#{param}</span>}})
        # 加上<a>标签
        step_name = step_match.format_args(lambda{|param| %{<span class="param"><a href="#{param}" target="_blank">#{param}</a></span>}})
        @builder.div(:class => 'step_name') do |div|
          @builder.span(keyword, :class => 'keyword')
          @builder.span(:class => 'step val') do |name|
            # name << h(step_name).gsub(/&lt;span class=&quot;(.*?)&quot;&gt;/, '<span class="\1">').gsub(/&lt;\/span&gt;/, '</span>’)
            # 包含 '.json’ 部分的 param 做 UrlEncode ，不包含的删去<a>标签
            if step_name.include? '.json'
              name << h(step_name).gsub(/&lt;span class=&quot;(.*?)&quot;&gt;/, '<span class="\1">').gsub(/&lt;\/span&gt;/, '</span>').gsub(/&lt;a href=&quot;(.*?)&quot; target=&quot;_blank&quot;&gt;/, '<a href="\1" target="_blank">').gsub(/&lt;\/a&gt;/, '</a>')
            else
              name << h(step_name).gsub(/&lt;span class=&quot;(.*?)&quot;&gt;/, '<span class="\1">').gsub(/&lt;\/span&gt;/, '</span>').gsub(/&lt;a href=&quot;(.*?)&quot; target=&quot;_blank&quot;&gt;/, '').gsub(/&lt;\/a&gt;/, '')
            end

          end
        end

        step_file = step_match.file_colon_line
        step_file.gsub(/^([^:]*\.rb):(\d*)/) do
          if ENV['TM_PROJECT_DIRECTORY']
            step_file = "<a href=\"txmt://open?url=file://#{File.expand_path($1)}&line=#{$2}\">#{$1}:#{$2}</a> "
          end
        end

        @builder.div(:class => 'step_file') do |div|
          @builder.span do
            @builder << step_file
          end
        end
      end

    end
  end
end
```
  
执行`cucumber -f Cucumber::Formatter::HtmlWithLink -o result.html`，新的报告里有链接了  
![](/images/2016/779106b264acc96cd0f2b8f7b5315020.png)  

另一种改法  
  
```ruby
require 'cucumber/formatter/html'

module Cucumber
  module Formatter
    class HtmlWithLink < Html
      def inline_js_content
        <<-EOF
  SCENARIOS = "h3[id^='scenario_'],h3[id^=background_]";
  $(document).ready(function() {
    $(SCENARIOS).css('cursor', 'pointer');
    $(SCENARIOS).click(function() {
      $(this).siblings().toggle(250);
    });
    $("#collapser").css('cursor', 'pointer');
    $("#collapser").click(function() {
      $(SCENARIOS).siblings().hide();
    });
    $("#expander").css('cursor', 'pointer');
    $("#expander").click(function() {
      $(SCENARIOS).siblings().show();
    });
    
  // 下面是新加的
  $("span.param").each(function(index,element){
    if ($(element).text().indexOf(".json") > 0) {
      text = $(element).text()
      href = "<a>" + text + "</a>"
      $(element).html(href)
      $(element).children("a").attr({
        "href" : text,
        "target" : "_blank"
      });
    }
  });
  // 上面是新加的

  })
  function moveProgressBar(percentDone) {
    $("cucumber-header").css('width', percentDone +"%");
  }
  function makeRed(element_id) {
    $('#'+element_id).css('background', '#C40D0D');
    $('#'+element_id).css('color', '#FFFFFF');
  }
  function makeYellow(element_id) {
    $('#'+element_id).css('background', '#FAF834');
    $('#'+element_id).css('color', '#000000');
  }

        EOF
      end
    end
  end
end
```
  

参考文章：[Is it possible to generate cucumber html reports with only scenario titles with out steps?](http://stackoverflow.com/questions/18271435/is-it-possible-to-generate-cucumber-html-reports-with-only-scenario-titles-with)  