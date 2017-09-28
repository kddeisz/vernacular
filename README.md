# Vernacular

[![Build Status](https://travis-ci.org/kddeisz/vernacular.svg?branch=master)](https://travis-ci.org/kddeisz/vernacular)
[![Gem Version](https://img.shields.io/gem/v/vernacular.svg)](https://rubygems.org/gems/vernacular)

Allows extending ruby's syntax and compilation process.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'vernacular'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install vernacular

## Usage

At the very beginning of your script or application, require `vernacular`. Then, configure your list of modifiers so that `vernacular` knows how to modify your code before it is compiled.

For example,

```ruby
Vernacular.configure do |config|
  pattern = /~n\(([\d\s+-\/*\(\)]+?)\)/
  modifier =
    Vernacular::RegexModifier.new(pattern) do |match|
      eval(match[3..-2])
    end
  config.add(modifier)
end
```

will extend Ruby syntax to allow `~n(...)` symbols which will evaluate the interior expression as one number. This reduces the number of objects and instructions allocated for a given segment of Ruby, which can improve performance and memory.

### `Modifiers`

Modifiers allow you to modify the source of the Ruby code before it is compiled by injecting themselves into the require chain through `RubyVM::InstructionSequence::load_iseq`.

### `RegexModifier`

Regex modifiers are by far the simpler of the two to configure. They take the same arguments as `String#gsub`. Either configure them with a string, as in:

```ruby
Vernacular::RegexModifier.new(/~u\((.+?)\)/, 'URI.parse("\1")')
```

or configure them using a block, as in:

```ruby
Vernacular::RegexModifier.new(pattern) do |match|
  eval(match[3..-2])
end
```

### `ASTModifier`

AST modifiers are somewhat more difficult to configure. A basic knowledge of the [`parser`](https://github.com/whitequark/parser) gem is required. First, extend the `Parser` to understand the additional syntax that you're trying to add. Second, extend the `Builder` with information about how to build s-expressions with your extra information. Finally, extend the `Rewriter` with code that will modify your extended AST by rewriting into a valid Ruby AST. An example is below:

```ruby
Vernacular::ASTModifier.new do |modifier|
  # Extend the parser to support and equal sign and a class path following the
  # declaration of a functions arguments to represent its return type.
  modifier.extend_parser(:f_arglist, 'f_arglist tEQL cpath', <<~PARSE)
    result = @builder.type_check_arglist(*val)
  PARSE

  # Extend the builder by adding a `type_check_arglist` function that will build
  # a new node type and place it at the end of the argument list.
  modifier.extend_builder(:type_check_arglist) do |arglist, equal, cpath|
    arglist << n(:type_check_arglist, [equal, cpath], nil)
  end

  # Extend the rewriter by adding an `on_def` callback, which will be called
  # whenever a `def` node is added to the AST. Then, loop through and find any
  # `type_check_arglist` nodes, and remove them. Finally, insert the
  # appropriate raises around the execution of the function to mirror the type
  # checking.
  modifier.build_rewriter do
    def on_def(node)
      type_check_node = node.children[1].children.last
      return super if !type_check_node || type_check_node.type != :type_check_arglist

      remove(type_check_node.children[0][1])
      remove(type_check_node.children[1].loc.expression)
      type = build_constant(type_check_node.children[1])

      @source_rewriter.transaction do
        insert_before(node.children[2].loc.expression, "result = begin\n")
        insert_after(node.children[2].loc.expression,
          "\nend\nraise \"Invalid return value, expected #{type}, " <<
          "got \#{result.class.name}\" unless result.is_a?(#{type})\nresult")
      end

      super
    end

    private

    def build_constant(node, suffix = nil)
      child_node, name = node.children
      new_name = suffix ? "#{name}::#{suffix}" : name
      child_node ? build_constant(child_node, new_name) : new_name
    end
  end
end
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake test` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/kddeisz/vernacular.

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
