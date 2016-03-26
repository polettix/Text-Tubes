# NAME

Data::Tubes - Text and data canalising

# VERSION

This document describes Data::Tubes version 0.01.

# SYNOPSIS

    use Data::Tubes qw< pipeline >;

    my $id = 0;
    my $tube = sequence(
       # automatic loading for simple cases
       'Source::iterate_files', # plugin to handle input files
       'Reader::by_line',       # plugin to read line by line
       'Parser::hashy',         # plugin to parse hashes

       # some operations will require some explicit coding of a tube
       # which is a sub ref with a contract on the return value
       sub {
          my $record = shift;
          $record->{structured}{id} = $id++;
          return $record;
       },

       # automatic loading, but with arguments
       [ # plugin to render stuff using Template::Perlish
          'Renderer::with_template_perlish',
          template => "[% a %]:\n  id: [% id %]\n  meet: [% b %]\n",
       ],
       [ # plugin to write stuff into output files, flexibly
          'Writer::to_files',
          filename => \*STDOUT,
          header   => "---\n",
          footer   => "...\n"
       ],

       # options for pipeline, in this case just pour into the sink
       {tap => 'sink'}
    );

    my $input = <<'END';
    a=Harry b=Sally
    a=Jekyll b=Hide
    a=Flavio b=Silvia
    a=some b=thing
    END
    $tube->([\$input]);

    ###############################################################

    # a somewhat similar example, with different facilities
    use Data::Tubes qw< drain summon >;

    # load components from relevant plugins
    summon(
       qw<
          +Plumbing::sequence
          +Source::iterate_files
          +Reader::read_by_line
          +Parser::parse_hashy
          +Renderer::render_with_template_perlish
          +Writer::write_to_files
          >
    );

    # define a sequence of tubes, they're just a bunch of sub references
    my $sequence = sequence(
       iterate_files(files => [\"n=Flavio|q=how are you\nn=X|q=Y"]),
       read_by_line(),
       parse_hashy(chunks_separator => '|'),
       render_with_template_perlish(template => "Hi [% n %], [% q %]?\n"),
       write_to_files(filename => \*STDOUT),
    );

    # run it, forget about what comes out of the end
    drain($sequence);

# DESCRIPTION

This module allows you to define and manage _tubes_, which are
transformation subroutines over records.

## First Things First: What's a _Tube_?

A sort of operative definition in code:

    my @outcome = $tube->($input_record);
    if (scalar(@outcome) == 0) {
       print "nothing came out, input record was digested!\n";
    }
    elsif (scalar(@outcome) == 1) {
       my $output_record = shift @outcome;
    }
    else {
       my ($type, $value) = @outcome;
       if ($type eq 'records') {
          my @output_records = @$value;
       }
       elsif ($type eq 'iterator') {
          while (my ($output_record) = $iterator->()) {}
       }
       else {
          die "sorry, this tube's output was not valid!\n";
       }
    }

A _tube_ is a reference to a subroutine that accepts a single, scalar
`$input_record` and can return zero, one or two (or more) values.

In particular:

- if it returns zero values, then the _tube_ just hasn't anything to
emit for that particular input record. The reasons depend on the tube,
but this is a perfectly valid outcome;
- if it returns one single value, that is the `$output_record`
corresponding to the `$input_record`. This is probably the most common
case;
- if it returns two (or more) values, the first one will tell you what is
returned (i.e. its _type_, and the second will be some way to get the
return value(s). This is what you would use if a single `$input_record`
can potentially give birth to multiple output records, like this:
    - -

        if you can/want to compute all the output records right away (e.g. you
        just to need to `split` something in the input record), you can use
        `records` for _type_ and pass a reference to an array as the second
        output value (each of them will be considered an output record);

    - -

        if you cannot (or don't want to) compute all the output records, e.g.
        because they might just blow out your process' memory, you can use
        _type_ `iterator` and return a subroutine reference back. This
        subroutine MUST be such that repeatingly calling it can yield two
        possible results:

        - o

            one single element, that is the _next_ output record, OR

        - o

            the empty list, that signals that the iterator has been emptied.

This is all that is assumed about tubes in the general case. Some
plugins will make further assumptions about what's expected as an input
record (e.g. a hash reference in most of the cases) or what is provided
as output records, but the generic case is all in the above definition.

A few examples will help at this point.

### A simple _filter_ tube

This is probably the most common type of tube: one record comes in, one
comes out. In the example, we will assume the input record is a string,
and will transform sequences of spacing characters into single spaces:

    my $tube = sub {
       my $text = shift;
       $text =~ s{\s+}{ }gmxs;
       return $text;
    };

### A `grep`-like tube

This is a tube that might potentially _digest_ the input record,
providing nothing out. In the example, we will assume that we're
focusing on valid non-negative integers only, and we will ignore
everything else:

    my $tube = sub {
       my $number = shift;

       # caution! A simple "return" is much more different than
       # "return undef", the first one is what we need to provide
       # "nothing" as output in the list context!
       return unless defined $number; # ignore input undef:s
       return unless $number =~ m{\A (?: 0 | [1-9]\d* ) \z}mxs;

       # this record passed all check, let's return it
       return $number;
    };

### A few little children out of your input

This is a tube that will typically generate a few output records from an
input one. It's best suited to be used when you know that you have
control over the number of output records, and they will not make your
memory consumption explode. In the example, we will provide "words" from
a text as output records:

    my $tube = sub {
       my $text = shift;
       my @words = split /\W+/mxs, $text;
       return (records => \@words);
    };

### Turning a filename into lines

This is a tube that might generate a lot of records out of a single
input one, so it's your best choice when you don't feel too confortable
with using the `records` alternative above. In the example, we will
turn an input file name into a sequence of lines from that file:

    my $tube = sub {
       my $filename = shift;
       open my $fh, '<', $filename or die "open('$filename'): $!";

       # the iterator is a reference to a sub, no input parameters
       my $iterator = sub {
          my ($line) = <$fh> or return;
          return $line;
       };
    };

## How Can Data::Tubes Help Me Then?

Data::Tubes can help you out in different ways:

- it provides you with a definition of tube (i.e. a _transforming
function_) that will help you control what you're doing. We already
talked about this format, just take a look at
["First Things First: What's a _Tube_"](#first-things-first-what-s-a-tube)
- it gives you some _plumbing_ facilities to easily perform some common
actions over tubes, e.g. put them in sequence or dispatch an input
record to the right tube. This is the kind of stuff that you can find in
[Data::Tubes::Plugin::Plumbing](https://metacpan.org/pod/Data::Tubes::Plugin::Plumbing);
- it gives you a library of pre-defined tube types that will help you with
common tasks related to transforming input data in output data (e.g. in
some kind of _Extract-Transform-Load_ process). This is what you can
find in the _Data::Tubes::Plugin_ namespace!

# FUNCTIONS

- **drain**

        drain($tube, @tube_inputs);

    drain whatever comes out of a tube. The tube is run with the provided
    inputs, and if an iterator comes out of it, it is repeatedly run until
    it provides no more output records.

- **summon**

        # Direct function import
        summon('Some::Package::subroutine');

        # DWIM, treat 'em as plugins under Data::Tubes::Plugin
        summon(
           {
              '+Source' => [ qw< iterate_array open_file > ],
           },
           [ qw< +Plumbing sequence logger > ],
           '+Reader::read_by_line',
        );

    summon operations, most likely from plugins.  This is pretty much the
    same as a regular `import` done by `use`, only supposed to be easier
    to use in a script.

    You can pass different things:

    - _array_

        the first item in the array will be considered the package name, the
        following ones sub names inside that package;

    - _hash_

        each key will be considered a package name, pointing to either a string
        (considered a sub name) or an array (each item considered a sub name);

    - _string_

        this will be considered a fully qualified sub name, i.e. including the
        package name at the beginning.

    In every case, if the package name starts with a `+` plus sign, the
    package name will be considered relative to `Data::Tubes::Plugin`, so
    the `+` plus sign will be substitued with `Data::Tubes::Plugin::`. For
    example:

        +Plumbing becomes Data::Tubes::Plugin::Plumbing
        +Reader   becomes Data::Tubes::Plugin::Reader

    and so on.

    It's probable that the `import` method will be overridden to make this
    import easy directly upon `use`-ing this module, instead of explicitly
    calling `summon`.

# BUGS AND LIMITATIONS

Report bugs either through RT or GitHub (patches welcome).

# AUTHOR

Flavio Poletti <polettix@cpan.org>

# COPYRIGHT AND LICENSE

Copyright (C) 2016 by Flavio Poletti <polettix@cpan.org>

This module is free software. You can redistribute it and/or modify it
under the terms of the Artistic License 2.0.

This program is distributed in the hope that it will be useful, but
without any warranty; without even the implied warranty of
merchantability or fitness for a particular purpose.

# POD ERRORS

Hey! **The above document had some coding errors, which are explained below:**

- Around line 278:

    You forgot a '=back' before '=head1'
