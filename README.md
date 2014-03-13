# Params - Ruby Library for handling HTTP params in Rails

## Why?

I reviewed the controller code of [Sharetribe](https://github.com/sharetribe/sharetribe) and I noticed a pattern: There are lots of code that handles Rails `params`. This code tends to be rather trivial yet error-prone and hard to follow due to long if-else chains.

There were couple action we were doing very often with the params:

1. Fetch a value from params hash (get)
1. Use default value, if parameter is not set (getOrElse)
1. Use another value from the params hash, if original parameter value is not set (getOrElse)
1. Convert parameter values (map)
1. Rename parameters (mv)
1. Remove parameters (rm)
1. Check if a single (nested) parameter was defined (defined?)
1. Check if any of the given (nested) parameters is defined (any?)
1. Check if all of the given (nested) parameters were present (all?)
1. Performing an action if given (nested) parameter is defined (with)

## Using Params library

### Wrapping and unwrapping
Params class is a wrapper that takes the `params` hash on initialization:

```ruby
params = {
  user: {
    name: {
      first: "Mikko"
    },
    location: {
      address: "Betonimiehenkuja 5",
      city: "Helsinki",
      country: "Finland"
    }
  },
  styles: {
    color1: "FF0000",
    "cover_photo": "cover.jpg"
  }
}

p = Params.new(params)
p.class.name    # => Params
```

The `p` is not hash anymore. It's an instance of Param class. 

Use `to_hash` to unwrap `p` back to a hash:

```ruby
h = p.to_hash
h.class.name    # => Hash

params == Params.new(params).to_hash      # => true, unwrapper hash and `params` have equal value
params.equal?(Params.new(params).to_hash) # => false, unwrapper hash is not the same object as the `params` hash
```

### `get(locator)`

Get a value from hash. If it's not set, throw exception.

```ruby
params.get(":user:location:address")   # => "Betonimiehenkuja 5"
params.get(":user:location")   # => { ... }
params.get(":user:location:zipcode")   # => throws
```

### `getOrElse(locator, default=nil)`

Get a value from hash. If it's not set, return `default` value. If `default` value is not provided, return `nil`. Never throw exception.

```ruby
params.getOrElse(":user:location:address", "Street address") # => "Betonimiehenkuja 5"
params.getOrElse(":user:location:zipcode", "00000")          # => "00000"
params.getOrElse(":user:location:zipcode", params.getOrElse(":user:location:address", "N/A")) # => "Betonimiehenkuja 5"
```

### `getOrElse(locator, [fallback_locator_1, ... , fallback_locator_n], default)`

Get a value from hash. If the value is not set, use the first fallback locator and return its value. If the value of first fallback locator is not set, use second, etc. If there are no values set for any given locator, return `default` value.

```
params.getOrElse(":user:location:zipcode", ":user:location:address", "N/A") # => "Betonimiehenkuja 5"
params.getOrElse(":user:location:zipcode", ":user:location:state", "N/A") # => "N/A"
params.getOrElse(":user:location:zipcode", ":user:location:state", ":user:location:country", "N/A") # => "Finland"
```

### `map(&block)`

Get a value from hash and map the value with `block`.

If value is not set, do nothing.

```ruby
mappedParams = params.map(":user:location:city", &:upcase)
mappedParams.get(:city) # => "HELSINKI"
```

### `defined?`

Return `true` if the value is set.

```ruby
has_address = params.defined?(":user:location:address")
has_address # => true
```

### `any?(&block)`

Return `true` if any of the values is set.

```ruby
needs_stylesheet_compilation = params.any?(:color1, :color2, :coverphoto, :small_cover_photo)
```

### `all?(&block)`

Return `true` if all of the values are set.

```ruby
can_show_fullname = params.all?(":user:name:first", ":user:name:last")
```

### `cp(source, destination)`

Copy value from a `source` location to `destination`

```ruby
copied = params.cp(":user:location:address", ":address")
copied.get(":address")               # => "Betonimiehenkuja 5"
copied.get(":user:location:address") # => "Betonimiehenkuja 5"
```

### `rm(locator)`

Remove a value.

```ruby
removed = params.rm(":user:location:address")
removed = params.get(":user:location:address") # => throws
```

### `mv(source, destination)`

Move a value.

This is a syntactic sugar for `params.cp(source, destination).rm(source)`

```ruby
moved = params.cp(":user:location:address", ":address")
moved.get(":address")               # => "Betonimiehenkuja 5"
moved.get(":user:location:address") # => throws
```

### `with(key, &block)`

Run a given block only if value is defined.

```ruby
params.with(":user:location:address") { |address| puts "Address: #{address}"} } # => "Address: Betonimiehenkuja 5"
params.with(":user:location:city") { |city| puts "City: #{city}"} } # => nothing
```

## Examples

### Supporting old URL params

Before:

```ruby
if !params[:view] && params[:map] == "true" then
  redirect_params = params.except(:map).merge({view: "map"})
  redirect_to url_for(redirect_params), status: :moved_permanently
end
```

After:

```ruby
Params.new(params).with(:map, "true") do |p, value|
  redirect_params = p.mapTo(:map, :view, "map")
  redirect_to url_for(redirect_params), status: :moved_permanently
end
```