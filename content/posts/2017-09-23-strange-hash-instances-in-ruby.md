---
title: "Strange Hash Instances in Ruby"
description: "Like (every?) Ruby user I contantly use the built-in Hash class..."
uuid: "b20b1985-0683-4b70-9b3c-d7c45decf236"
date: 2017-09-23
slug: "strange-hash-instances-in-ruby"
tags: ["ruby", "star"]
categories: ["star", "rss"]
image: "images/posts/hash-monkey-patching.png"
image_width: 728
image_height: 728
---
Note: All code was run using Ruby MRI 2.4.1 and is not guaranteed to behave the same in other implementations (JRuby, mruby, etc). *Also you probably don't want to do this in a real project.*

<hr />

Like (every?) user of Ruby I constantly use the built-in `Hash` class. I've made my own classes usable as keys and know that there are [multiple equality methods](https://ruby-doc.org/core-2.4.2/Object.html#method-i-eql-3F). With this understanding I decided to try and make strange `Hash` instances.

## Strange Hash #1: Non-unique keys

How about a hash that only allows one value for any `Integer` key regardless of the integer value? From the documentation it seems that this is as simple as changing `Integer#hash` to always return the same value and `Integer#eql?` to always return true:

{{< highlight ruby >}}
class Integer
   def eql?(other)
     true
   end

  def hash
     0
  end
end

table = {}
table[1] = 'one'
table[5] = 'five'
puts table
#=> {1=>"one", 5=>"five"}
{{< /highlight >}}

Strange, that wasn't what I expected. The table has entries for both `1` and `5` even though I don't expect `Hash` to be able to differentiate these keys anymore. Playing around with some of these monkey-patched integers shows them working how I expect:

{{< highlight ruby >}}
irb(main)> 1 == 2
=> false
irb(main)> 1.eql? 2
=> true
irb(main)> 1.hash
=> 0
irb(main)> 2.hash
=> 0
{{< /highlight >}}

I know I've defined `eql?` and `hash` on my own classes before and had it work the way the Ruby documentation describes. How about trying the same patch but for the Array class instead?

{{< highlight ruby >}}
class Array
  def eql?(other)
    true
  end

  def hash
    0
  end
end

table = {}
table[[1]] = 'array of one'
table[[5]] = 'array of five'
puts table
#=> {[1]=>"array with five"}
{{< /highlight >}}

Both `[1]` and `[5]` were treated as the same key! `table[[5]] = 'array with five'` caused the value for `[1]` to be over-written. Any lookup with an array key will return the last value stored in `table` with an array key 🎉

{{< highlight ruby >}}
irb(main)> table[[1]]
=> "array with five"
irb(main)> table[[5]]
=> "array with five"
irb(main)> table[['hello']]
=> "array with five"
{{< /highlight >}}

### Solving the Mystery

At this point I wasn't satisfied with my understanding. Why does monkey-patching `Array` work when `Integer` doesn't? Some more testing shows that other classes like `Symbol` and `String` also stubbornly continue to be treated as unique keys with the monkey-patch. The documentation doesn't mention any special cases and the monkey patch works, just not when used as a key for a `Hash`.

Having tried documentation and experimentation I decide to move on to my next favourite option: looking at the MRI source code! Some relevant code in [hash.c](https://github.com/ruby/ruby/blob/820605ba3c/hash.c#L99-L116) jumps out:

{{< highlight c >}}
rb_any_cmp(VALUE a, VALUE b) {
  // ... snip
  if (FIXNUM_P(a) && FIXNUM_P(b)) {
    return a != b;
  }
  if (RB_TYPE_P(a, T_STRING) && RBASIC(a)->klass == rb_cString &&
      RB_TYPE_P(b, T_STRING) && RBASIC(b)->klass == rb_cString) {
    return rb_str_hash_cmp(a, b);
  }
  // ... snip
  if (SYMBOL_P(a) && SYMBOL_P(b)) {
    return a != b;
  }
  // ... snip
{{< /highlight >}}

This certainly looks like optimizations for handling string, number and symbol keys. There is also some similar code in the [`any_hash` function](https://github.com/ruby/ruby/blob/820605ba3c/hash.c#L171-L199). Now that I know these are special cases I can steer clear when creating strange `Hash` instances.

## Strange Hash #2: Duplicate Keys

How about storing multiple values for the same key? If an object returns inconsistent values for `hash` then the `Hash` class won't consider it to be the same object:

{{< highlight ruby >}}
class Array
  @@last_id = 0

  def eql?(other)
    false
  end

  def hash
    @@last_id = @@last_id + 1
  end
end

table = {}
table[[0]] = 'array of zero'
table[[0]] = 'another array of zero'

puts table
#=> {[0]=>"array of zero", [0]=>"another array of zero"}
puts table.keys
#=> [[0], [0]]
{{< /highlight >}}

As expected `table` has multiple entries with for a single key 🎉

However getting back values stored using `Array` keys no longer works:

{{< highlight ruby >}}
table = {}
table[[0]] = 'array of zero'
puts table[[0]]
#=> nil
{{< /highlight >}}

## Strange Hash #3: Expiring Key

{{< highlight ruby >}}
class Array
  def eql?(other)
    Time.now.to_i < (@@key_expires_at || 0)
  end

  def hash
    @@key_expires_at ||= Time.now.to_i + 3
  end
end

table = {}
table[['expires']] = 'this value is only available for 3 seconds'
puts table[['expires']]
#=> 'this value is only available for 3 seconds'

sleep(5)
puts table[['expires']]
#=> nil
{{< /highlight >}}

Now an array can only be used to retrieve a value within a set time period. This also causes all array keys will be seen as identical just like the first example.

## Future Fun

I would love to be able to modify `Array#hash` and `Array#eql?` to behave differently when called by `Hash` for insertion vs. retrieval. Unfortunately I don't see anything in `caller` to let me differentiate.
