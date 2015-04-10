<!-- maruku -o hashlib.html hashlib.md -->

<style type="text/css">
pre code { display: block; margin-left: 2em; }
div { display: block; margin-left: 2em; }
ins { text-decoration: none; font-weight: bold; background-color: #A0FFA0 }
del { text-decoration: line-through; background-color: #FFA0A0 }
</style>

<table><tbody>
<tr><th>Doc. no.:</th>	<td>Dnnnn</td></tr>
<tr><th>Date:</th>	<td>2015-04-09</td></tr>
<tr><th>Project:</th>	<td>Programming Language C++, Library Evolution Working Group</td></tr>
<tr><th>Reply-to:</th>	<td>Zhihao Yuan &lt;zhihao.yuan at rackspace dot com&gt;</td></tr>
</tbody></table>

# Message Digest Library for C++

## Motivation

Cryptographic hash functions hold irreplaceable roles in a large variety of
applications, since security and data integrity are topics that cannot be
dismissed to the applications involving data exchanging.  Being compared to
the ordinary hash functions being used with the unordered containers, the
cryptographic functions are significantly different in purposes and design,
which result in different properties on interface, implementation, and
performance.  In order to satisfy such demands in C++, these functions should
be standardized as a part of the C++ standard library.

## Scope

This paper focuses on exposing the core functionalities of the cryptographic
hash functions in an easy-to-use and efficient way; some use cases that can be
provided in a higher level abstraction are not covered, including but not
limited to Base64 digest, fingerprinting user-defined types, and key
derivation.

This paper does not discuss `std::hash`, because to co-work with the unordered
containers is not a main purpose of cryptographic hash functions.

This paper has no direct relationship with
[N3980](http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2014/n3980.html),
but some functionalities described here can serve as the underlying
implementation of a `HashAlgorithm`.  More details can be found in
[Design Decisions](#design_decisions).

## Examples

    namespace hashlib = std::hashlib;

    hashlib::sha1 h;

    assert(h.hexdigest() == "da39a3ee5e6b4b0d3255bfef95601890afd80709");

    h.update("C-string");
    h.update(std::string());

    auto h2 = h;

    assert(h2 == h);                    // hasher is equationally complete
    assert(h2.digest() == h.digest());  // semantics same as above

    h.update(buf, sizeof(buf));         // sized input

    assert(h.digest() == h.digest());   // can get result multiple times

    using rmd160 = hashlib::hasher<rmd160_provider>;  // user-defined

## Design Decisions

### Interface

The design is unashamedly stolen from Python's `hashlib` module`[1]`.  The
details are as follows:

- Provide both incremental hashing interface and one-shot hashing
  interface.  They accept the same types of input.

- No `Callable` interface.  There are two conflicting options here:

  1. `operator()` performs one-shot hashing and returns the digest.  This
     matches the mathematical definition of a hash function, but the input
     has to be stored contiguously in memory.

  2. `operator()` performs incremental hashing, as an unnamed `update`.
     But different from a `HashAlgorithm` in N3980`[2]`, the users of a
     cryptographic hash function are often end-users instead of library
     authors, where a named method is more clear.

  Considering the widest audiences, neither use is more important than the
  other, and both can be easily solved with lambdas:

<div><div><pre>
[](auto&amp;&amp;... ts) { return hashlib::sha1(ts...).digest(); }
[&amp;](auto&amp;&amp;... ts) { h.update(ts...); }
</pre></div></div>

- Support returning the result in hexadecimal.  Returning `std::string` is not
  the most efficient interface to serialize the digest, but it's very
  convenient to many users.

- The result can be retrived multiple times.  This is a highly desirable
  feature to cryptographic hash functions.  Because

  1. it makes hasher types more *Regular* by enabling `EqualityComparable`
     with the desired behavior -- comparing the digest.  Other options are
     not so satisfactory to me, including defining no `operator==`, which makes
     `sha1(a) == sha1(b)` not compilable, or providing a modifying
     `operator==`, which is a non-option since `h != h`, or comparing the
     underlying hash contexts, which is technically viable but logically hard
     to explain and hard to verify.

  2. it simplifies some specific use cases.  For example,

<div><div><div><pre>
log(h);                   // user can insert a log line without making
v.push_back(h.digest());  // the rest of the code UB
                          // and the log line does not need to be log(sha1(h))
</pre></div></div></div>


>> Or, if one has a hash value of a file block with a known starting point in
>> a file, and we want to know the size of the block, we can retrieve and
>> compare the digest for each byte we consumed.

> This goal is achieved by finalizing the hash on a copy of the underlying hash
> context.  The state sizes of the cryptographic hash functions are
> constrained`[3]` (actual context sizes: 96 for SHA1, 208 or 216 for SHA512)
> -- even hashing 1 byte is more costly than copying the context.
> Considering the aforementioned benefits, I claim that the "overhead" is
> worthy.

### Extensibility

The rich interface introduced above is featured by the class template
`hashlib::hasher<HashProvider>`.  A user can add a new hasher by instantiating
the template with a struct satisfying the `HashProvider` requirements by
describing the implementation of the hash function in terms of a
`context_type` and some core operations involving such a type.

The details are discussed in
[Technical Specifications](#technical_specifications).

### Choice of algorithms

I propose to require the following message digest algorithms in the standard:

- MD5
- SHA1
- SHA256
- SHA512

Standard library implementations need to make all the required algorithms
available.  If an implementation can achieve this by shipping a header and
rely on the base system libraries, we can save an enormous effort to implement
and to maintain those algorithms.  The design of the message digest library in
this proposal made this approach easier, and the choice of algorithms also took
the algorithm availability on major platforms into consideration (the last 3
are for reference):

&nbsp; | GNU(\*)  | FreeBSD  | Apple        | Windows   | Android  | Java     | .Net     | Python
------ | -------- | -------- | ------------ | --------- | -------- | -------- | -------- | --------
&nbsp; | libc     | libmd    | CommonCrypto | CryptoAPI | OpenSSL  |          |          | &nbsp;
MD5    | &#x2713; | &#x2713; | &#x2713;     | &#x2713;  | &#x2713; | &#x2713; | &#x2713; | &#x2713;
SHA1   | &#x2713; | &#x2713; | &#x2713;     | &#x2713;  | &#x2713; | &#x2713; | &#x2713; | &#x2713;
SHA224 |          |          | &#x2713;     |           | &#x2713; |          |          | &#x2713;
SHA256 | &#x2713; | &#x2713; | &#x2713;     | &#x2713;  | &#x2713; | &#x2713; | &#x2713; | &#x2713;
SHA384 |          |          | &#x2713;     | &#x2713;  | &#x2713; |          | &#x2713; | &#x2713;
SHA512 | &#x2713; | &#x2713; | &#x2713;     | &#x2713;  | &#x2713; |          | &#x2713; | &#x2713;
RMD160 |          | &#x2713; |              |           |          |          | &#x2713; | 

\* These functions are private to `crypt(3)`.

MD5 is included for supporting existing services; any algorithms weaker than
that are excluded.

Based on the research done so far, I believe that the aforementioned 4
algorithms are the most important ones which should be required to support,
and possible to support (without duplicated work) on the major platforms.

## Technical Specifications

This section gives the layout of a wording.

> ## Message digests

> This subclause defines a class template `hasher` as a common interface to
> the cryptographic hash and message digest algorithms, and the typedefs
> `sha1`, `sha256`, `sha512`, and `md5` for the unspecified specializations of
> `hasher` to implement, respectively, the FIPS secure hash algorithms SHA1,
> SHA256, and SHA512 \[FIPS 180-2] as well as RSA's MD5 algorithm \[RFC 1321].

> Through out this subclause, to specialize a template with a template type
> parameter named `HashProvider`, the corresponding template argument shall
> meet the `HashProvider` requirements.

> ### `HashProvider` requirements

> ### Header `<hashlib>` synopsis

    namespace std::hashlib {  // N4026

      template <typename HashProvider>
      struct hasher
      {
        static constexpr auto digest_size = HashProvider::digest_size;
        static constexpr auto block_size  = HashProvider::block_size;

        typedef typename HashProvider::context_type      context_type;
        typedef std::array<unsigned char, digest_size>   digest_type;

        hasher();

        explicit hasher(char const* s);
        explicit hasher(char const* s, size_t n);

        template <typename StringLike>
          explicit hasher(StringLike const& bytes);

        void update(char const* s);
        void update(char const* s, size_t n);

        template <typename StringLike>
          void update(StringLike const& bytes);

        digest_type digest() const;

        template <typename CharT = char,
                  typename Traits = char_traits<CharT>,
                  typename Allocator = allocator<CharT>>
          basic_string<CharT, Traits, Allocator>
            hexdigest() const;

      private:
        context_type ctx_;  // exposition only
      };

      template <typename HashProvider>
      bool operator==(hasher<HashProvider> const& a,
                      hasher<HashProvider> const& b);

      template <typename HashProvider>
      bool operator!=(hasher<HashProvider> const& a,
                      hasher<HashProvider> const& b);

      template <typename CharT, typename Traits,
                typename HashProvider>
        basic_ostream<CharT, Traits>&
          operator<<(basic_ostream<CharT, Traits>& out,
                     hasher<HashProvider> const& h);

      typedef hasher<unspecified>   md5;
      typedef hasher<unspecified>   sha1;
      typedef hasher<unspecified>   sha256;
      typedef hasher<unspecified>   sha512;

    }

## Implementation

A prototype is available at
<https://github.com/lichray/cpp-deuceclient/blob/master/include/deuceclient/hashlib.h>.

## References

`[1]` "`hashlib` -- Secure hashes and message digests."
      _The Python Standard Library_.
      <https://docs.python.org/2/library/hashlib.html>

`[2]` Hinnant, Howard E., Vinnie Falco, and John Bytheway.
      "Proposed Wording." _Types Don't Know #_.
      <http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2014/n3980.html#wording>

`[3]` "SHA-1."
      _Wikipedia: The Free Encyclopedia_.
      <http://en.wikipedia.org/wiki/SHA-1#Comparison_of_SHA_functions>
