# marxist.org-file-test
#!/usr/bin/perl -w

#======================#
# By Trevor Schroeder
# Version 2.5
# 02/09/2002
# Licensed under the terms of the GNU General Public License version 2.0
# or greater.
#======================#

# Diddle this to make it so that all intros, prefaces, and the like are
# output as chapters less than 1.  For example, 0 is appropriate for a
# text that launches right into the chapters.
$STARTING_CHAPTER=0;

# What are the files to be named?
$FILENAMES='ch%CHAPTER%.htm';

# Duh.
$INDEXNAME='index.htm';

# At least how many blank lines lead up to a title
$TITLE_SEP_COUNT = 2;

# Default location of works.css
$DEFAULT_WORKSREF='../../../../../css/works-blue.css';

# If we come across something that *looks* like a subtitle, ubt is
# longer than this, it must not be.
$MAX_SUBTITLE_LEN=160;

# How many lines do we want to be able to remember.  Leave this alone if
# you're not sure what it's for.
#
# Aside:  this *was* going to be used for history keeping, but instead I'm
# using seeks back and forth when that's necessary.  If we ever have a
# problem with using seek, I'll resurrect the idea of using a history
# window.  For now I leave it in because it really doesn't cost that much.
$WINDOW_SIZE=5;

if($#ARGV!=0) { usage(\*STDERR); exit(1) };

open(DATAFILE, "< $ARGV[0]") or die "$ARGV[0]: $!";

# Pick one or the other, depending on whether you want to have the info at
# the top of the file being marked up or if you want to simply enter it by
# hand.
&read_info(\*STDIN, \*STDOUT);
#&read_info(\*DATAFILE);

$current_chapter=$STARTING_CHAPTER;
# Make it so that we start by thinking the first thing we come across
# that's non-blank is a title.  This is probably correct.
$blank_count=$TITLE_SEP_COUNT;
# Prime the pump by opening an introduction
&open_chapter(\*HTMLFILE,$current_chapter,"Introduction");
# Stuff the last_few array
for (0..$WINDOW_SIZE-1)
{ $last_few[$_]=""; }
$footnote_count=0;
$last_written_footnote=0;
###################
# Do the fun stuff!
###################
while($line=<DATAFILE>)
{
	# Shift the oldest entry out of the window and put the newest in
	shift @last_few;
	@last_few = (@last_few, $line);

	# Is this a chapter title?
	if(($blank_count > $TITLE_SEP_COUNT) and ! ($line =~ /^\s*$/))
	{
		print "# New chapter: \n";
		$current_chapter++;
		($chapter_title,$line) = &read_title(\*DATAFILE,$line);

		print "$chapter_title\n\n";

		# Record it in the TOC array
		$chapter_titles["$current_chapter"]=$chapter_title;

		&close_chapter(\*HTMLFILE,$current_chapter,$chapter_title);
		# Open the current chapter file and write out the header
		&open_chapter(\*HTMLFILE,$current_chapter,$chapter_title);

		# We set it to -1 so that at the end, when $line =~ /^$/
		# it will get upped to 0.
		$blank_count=-1;
		@footnotes=();
		$footnote_count=0;
	}

	# Footnote?
	elsif($line =~ /^\s*\[[0-9]+\]\s+(.+)$/)
	{
		&record_footnote(\*DATAFILE,$line);
		# Do similar hackery as done in the chapter changing code
		$line="";
		$blank_count=-1;
	}

	# </P> ?
	if( (! ($last_few[$WINDOW_SIZE-2] =~ /^[ ]*$/) and
				($line =~ /^[ ]*$/)) ||
	    (($last_few[$WINDOW_SIZE-2] =~ /(\.|\")\s*$/) and
	     			($line =~ /^([ ]{3,}|\t)/)) )
	{ print HTMLFILE "</p>\n"; }
        # <P> ?
	if( (($last_few[$WINDOW_SIZE-2] =~ /^[ ]*$/) and
				! ($line =~ /^[ ]*$/)) ||
	    (($last_few[$WINDOW_SIZE-2] =~ /(\.|\")\s*$/) and
	     			($line =~ /^([ ]{3,}|\t)/)) )
	{ print HTMLFILE "<p>\n"; }

	# Make refs to the footnotes
	if($line =~ /.+\[([0-9]+)\]/)
	{ $line = &mark_footnote($1, $line); }


	# Count up consectutive blank lines
	if($line =~ /^\s*$/)
	{ $blank_count++; }
	else
	{ $blank_count=0; }

	print HTMLFILE $line;

}
# Close the file and stick in the footer
&close_chapter(\*HTMLFILE);

&make_index;

########################
### END OF MAIN BODY ###
########################

# This converts an inline footnote reference to appropriate HTML
# The first arg is the number of the footnote.  The second is the line to
# do the search and replace on.
sub mark_footnote
{
	my $number = shift;
	my $line = shift;

	my $note="<sup class=\"anote\"><a href=\"#$1\" name=\"${1}b\">($1)"
		."</a></sup>";
	$line =~ s/\[([0-9]+)\]/$note/;
	return $line;
}

# This records a footnote in the footnote array.  The first arg is a
# source filehandle ref, the second is the footnote so far.
sub record_footnote
{
	my $FILE = shift;
	my $line = shift;

	$footnote_count++;
	while(not $line =~ /^$/)
	{
		$footnotes[$footnote_count] .= $line." ";
		$line=<$FILE>;
	}
	# Strip off the comment number.
	$footnotes[$footnote_count] =~ s/^(\[[0-9]+\])\s+(.+)$/$2/m;
}


# This simply reads a title.  The first arg is a filehandle ref the second
# is the title so far.
sub read_title
{
	my $FILE = shift;
	my $chapter_title = shift;
	my $chapter_subtitle = "";

	$_=<$FILE>;

	# Read main title
	while(defined $_ && not $_ =~ /^\s*$/)
	{
		$chapter_title .= $_;
		$_=<$FILE>;
	}

	# Prepare to do voodoo seek and lookahead magic.
	my $curpos = tell($FILE);

	while(defined $_ && $_ =~ /^\s*$/)
	{ $_=<$FILE>; }

	# Now read potential subtitles.
	while(defined $_ && not $_ =~ /^\s*$/)
	{
		$chapter_subtitle .= $_;
		$rewpos=tell($FILE);
		$_=<$FILE>;
	}

	chomp($chapter_title);
	chomp($chapter_subtitle);

	# If this is NOT a subtitle...
	# if( ($chapter_subtitle =~ /^.*(\.|\"|\?|\!|\])\s*$/m)
	# The colon may or may not be a good idae...
	if( ($chapter_subtitle =~ /^.*(\.|\"|\?|\!|\]|\:|\,)\s*$/m)
          ||(length($chapter_subtitle) > $MAX_SUBTITLE_LEN))
	{
		# Here's hoping this is a real file and not an I/O stream
		seek($FILE,$curpos,0);
	}
	else
	{
		$chapter_title .= " (".$chapter_subtitle.")";
		# Rewind to the blank just after the title
		seek($FILE,$rewpos,0);
	}

	return ($chapter_title,$_);
}

# This will expand filenames.  Pass the number of the chapter as the first
# argument
sub filename
{
	my $CHAPTER_NUM = shift;
	(my $file_name = $FILENAMES) =~ s/\%CHAPTER\%/$CHAPTER_NUM/g;
	if($CHAPTER_NUM =~ /[0-9]+/ && $CHAPTER_NUM < 10)
	{ $file_name =~ s/(\D+)(\d)(\D+)/${1}0$2$3/g; }
	return $file_name;
}


# This will open the index file and stick in the header
sub open_index
{
	my $FILE = shift;
	my $TITLE = shift;
	my $current_file = $INDEXNAME;

	open($FILE, "> $current_file") or die "$current_file: $!";

	################
	# Static data!!
	################
	print $FILE qq(<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
        "http://www.w3.org/TR/2000/REC-xhtml1-20000126/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
<head>
   <meta http-equiv="content-type" content="text/html; charset=iso-8859-1" />
   <meta name="author" content="$info{'Author'}" />
   <title>$TITLE</title>
   <link rel="stylesheet" type="text/css" href="../../../../../css/border-blue.css" />
</head>

<body>

<blockquote>
 <div class="border">

  <h2>$info{'Author'}</h2>

);
	if(length($info{'Title'}) < 12)
	{ print $FILE "<h1>$info{'Title'}</h1>\n"; }
	else
	{ print $FILE "<h3>$info{'Title'}</h3>\n"; }

}

# This will open the newest chapter file and stick in the header
sub open_chapter
{
	my $FILE = shift;
	my $CHAPTER_NUM = shift;
	my $CHAPTER_NAME = shift;
	my $current_file = &filename($CHAPTER_NUM);

	open($FILE, "> $current_file") or die "$current_file: $!";
	&print_header($FILE, $CHAPTER_NUM);

	################
	# Static data!!
	################
	print $FILE qq(

<h3>$CHAPTER_NAME</h3>

<hr />

<p>

);

}


# This will stick the footer in the chapter and close the file
sub close_chapter
{
	my $FILE = shift;

	# See if this chapter has refs to any following or not...
	if ($#_ == -1 )
	{ $NO_NEXT_CHAPTER = 1; }
	else
	{ $NO_NEXT_CHAPTER = 0; }

	my $CHAPTER_NUM = shift;
	my $CHAPTER_NAME = shift;

	# This will suppress some warnings
	if($NO_NEXT_CHAPTER) { $CHAPTER_NUM = 0; }

	my $next_file = &filename($CHAPTER_NUM);

	################
	# Static data!!
	################
	if($footnote_count > $last_written_footnote)
	{
		print $FILE qq(

<hr />

);
		for($i=$last_written_footnote+1;$i<=$footnote_count;$i++)
		{
			print $FILE qq(
<p class="fst">
<sup class="anote"><a href="#${i}b" name="$i">($i)</a></sup>
$footnotes[$i]
</p>
);
		}
		print $FILE qq(
<p class="skip">&#160;</p>
<hr />
);
		$last_written_footnote=$footnote_count;
	}

	unless ($NO_NEXT_CHAPTER)
	{
		print $FILE qq(
<p class="next">
Next: <a href="$next_file">$CHAPTER_NAME</a>
</p>
);
	}
	&print_footer($FILE);
	close($FILE);
}

# This will collect information about a particular work being transcribed.
# It takes two args: an INPUT filehandle and an OUTPUT filehandle ref (in
# that order, of course).
#
# If you want to add more fields, simply follow the general format used
# below.
sub read_info
{
	my $INPUT = shift;

	if($#_ == -1)
	{ $NO_OUTPUT = 1; }
	else
	{ $NO_OUTPUT = 0; }

	my $OUTPUT = shift;
	

	unless ($NO_OUTPUT) { print $OUTPUT "Book Title -> "; }
	chomp($info{"Title"}=<$INPUT>);

	unless ($NO_OUTPUT) { print $OUTPUT "Author -> "; }
	chomp($info{"Author"}=<$INPUT>);

	unless ($NO_OUTPUT) { print $OUTPUT "Date Written -> "; }
	chomp($info{"Written"}=<$INPUT>);

	unless ($NO_OUTPUT) { print $OUTPUT "Publication Date -> "; }
	chomp($info{"First Published"}=<$INPUT>);

	unless ($NO_OUTPUT) { print $OUTPUT "Source Information -> "; }
	chomp($info{"Source"}=<$INPUT>);

	unless ($NO_OUTPUT) { print $OUTPUT "Your Name -> "; }
	chomp($info{"Transcription"}=<$INPUT>);

	unless ($NO_OUTPUT)
	{
		print $OUTPUT "Hit return to accept the default, otherwise type the location for works css:"
			." [$DEFAULT_WORKSREF]\n -> ";
	}
	chomp($info{"worksref"}=<$INPUT>);
	if($info{"worksref"} =~ /^$/)
	{ $info{"worksref"} = $DEFAULT_WORKSREF; }

	$info{"Online Version"}=
		"$info{'Author'} Internet Archive (marxists.org) 2002.";

	print STDERR
		"        Title ... $info{'Title'}\n",
		"       Author ... $info{'Author'}\n",
		"      Written ... $info{'Written'}\n",
		"    Published ... $info{'First Published'}\n",
		"       Source ... $info{'Source'}\n",
		"Transcription ... $info{'Transcription'}\n",
		"     CSS Link ... $info{'worksref'}\n";

}

sub print_header
{
	my $FILE = shift;
	my $CHAPTER_NUM = shift;

	################
	# Chapter Static data!!
	################
	print $FILE qq(<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
        "http://www.w3.org/TR/2000/REC-xhtml1-20000126/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
<head>
   <meta http-equiv="content-type" content="text/html; charset=iso-8859-1" />
   <meta name="author" content="$info{'Author'}" />
   <title>$info{'Title'} &#151; Ch $CHAPTER_NUM</title>
   <link rel="stylesheet" type="text/css" href="$info{'worksref'}" />
</head>
	
<body>

<p class="title">
$info{'Author'}
</p>

<hr />
);

}

sub close_index
{
	my $FILE = shift;

	################
	# Static data!!
	################
	print $FILE qq(

<hr />

<p class="footer">
<a href="">$info{'Author'} Internet Archive</a>
</p>

</div>
</blockquote>

</body>
</html>

);

}

sub print_footer
{
	my $FILE = shift;

	################
	# Static data!!
	################
	print $FILE qq(

<hr />

<p class="footer">
Table of Contents: <a href="index.htm">$info{'Title'}</a>
</p>

</body>
</html>

);

}

# Print usage string.
sub usage
{
	my $OUTPUT = shift;
	print $OUTPUT "Usage: $0 <filename>\n";
}

# This creates an index.html.
sub make_index
{
	&open_index(\*INDEXFILE,$info{"Title"});

	print INDEXFILE qq(
<hr />

<p class="information">
);

	print INDEXFILE "<span class=\"info\">Written:</span> "
		.$info{'Written'}."\n<br />\n";
	print INDEXFILE "<span class=\"info\">First Published:</span> "
		.$info{'First Published'}."\n<br />\n";
	print INDEXFILE "<span class=\"info\">Source:</span> "
		.$info{'Source'}."\n<br />\n";
	print INDEXFILE "<span class=\"info\">Transcription:</span> "
		.$info{'Transcription'}."\n<br />\n";
	print INDEXFILE "<span class=\"info\">Copyleft:</span> "
		.$info{'Online Version'}. "<span class=\"date\">Permission is granted to copy and/or distribute this document under the terms of the <a href=\"../../../../../admin/legal/fdl.htm\">GNU Free Documentation License</a>.</span>\n<br />\n";

	print INDEXFILE qq(
</p>

<hr />

<p class="toc">
Contents:
</p>
<p class="index">
);
	# Remember, the starting chapter is a dummy chapter and thus has
	# no title...
	for($i=$STARTING_CHAPTER+1;$i<=$current_chapter;$i++)
	{
		print INDEXFILE "<a href=\"".&filename($i)."\">"
			.$chapter_titles[$i]."</a><br />\n";
	}
	print INDEXFILE "</p>\n";

	&close_index(\*INDEXFILE);
}
