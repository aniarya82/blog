---
layout: post
title: First Commit
excerpt: "About my first commit as a part of GSoC 2016"
modified: 2016-05-13
categories: blog
tags: [gsoc2016, github]
image:
  feature: gsoc.png
  credit: Google
  creditlink: http://summerofcode.withgoogle.com/
comments: true
share: true
---

# First Commit
It was simple task to do. The current manner of downloading the xml, tsk files were to create tmp files and as soon as the downloads are over the tmp files get deleted. But now the user wanted to download the files again. So again it was generated in tmp files and the process goes on.
Now this process was long and taking up time and resources of the server. To free this up we thought to save the xml and tsv files and when user again wanted to download them, we could just give them the link of the file, no need to regenerate it.

To begin this, i began from `routes.rb`
{% highlight css %}
get '/download/:jid.:type' do |jid, type|
  job = Job.fetch(jid)
  out = BLAST::Formatter.new(job.rfile, type)
  # send_file out.file.path, :filename => out.filename, :type => out.mime
  send_file out.file, :filename => out.filename, :type => out.mime
end
{% endhighlight %}

`BLAST::Formatter:new(job.rfile, type)`  This was resposible to call formatter.rb where the run method was called to generate the xml, tsv files

{% highlight css %}
command =
 "blast_formatter -archive '#{archive_file}'" \
 " -outfmt '#{format} #{specifiers}'" \
 " -out '#{file.path}' 2> /dev/null"
logger.debug("Executing: #{command}")
Dir.chdir(File.exist?(DOTDIR) && DOTDIR || Dir.pwd) do
 system(command)
end
{% endhighlight %}

Here `file ||= Tempfile.new file` was creating a tmp file and command was creating specific type of file from the blast archive file and storing in this new tmp file.
So i went around and made some changes
created new variable `xfile = File.join(File.dirname(archive_file), filename)`. This will basically do is get the directory name of the archive_file and concatenate it with filename which i want for xml and tsv files.
Then the above method `def run` an addition is made to check whether the file already exists or not? If exist then return it. If not then call the command and execute it to generate the file and then return it.
But it sounds so easy right but it wasn’t.
Here when i changed the variables, file was a class and had a `.path` method but in the variable i created it was just a string. So i had to change  somethings in some places.
{% highlight css %}
send_file out.file.path, :filename => out.filename, :type => out.mime
{% endhighlight %}
Like this line in `routes.rb` changed to
{% highlight css %}
send_file out.file, :filename => out.filename, :type => out.mime
{% endhighlight %}
Similarly in `formatter.rb`
{% highlight css %}
command =
          "blast_formatter -archive '#{archive_file}'" \
          " -outfmt '#{format} #{specifiers}'" \
          " -out '#{file.path}’ 2> /dev/null"
{% endhighlight %}
changes to
{% highlight css %}
command =
          "blast_formatter -archive '#{archive_file}'" \
          " -outfmt '#{format} #{specifiers}'" \
          " -out '#{xfile}' 2> /dev/null"
{% endhighlight %}
I thought these will be the sufficient changes for the task to be done.
But these didn’t seen quite right there was a parsing error.

{% highlight css %}
ir = node_to_array(Ox.parse(xml.open.read).root)
{% endhighlight %}

This line in `report.rb` was having a issue `root not defined`. Then I noticed that `xml` variable was storing the wrong file name
{% highlight css %}
xml = Formatter.run(job.rfile, 'xml').file
{% endhighlight %}
This was calling the `file` method in Formatter class. But the new variable was stored in `xfile` method so this changed to
{% highlight css %}
xml = Formatter.run(job.rfile, 'xml').xfile
{% endhighlight %}
But this didn’t solved the error, instead an another error popped up.
`No open method defined on xml`
So i changed it to
{% highlight css %}
ir = node_to_array(Ox.parse(File.read(xml)))
{% endhighlight %}
But now another error popped up,
{% highlight css %}
NoMethodError - undefined method 'name' for #<Ox::DocType:0x007fd722ae53c0>:
{% endhighlight %}
This was for line 164 in `report.rb` for node_to_value method.

For now i am still stuck on this one.

Okay so i had a little chat with Priyam. He told me that `report.rb` was expecting File type and i was providing it String type, hence so many errors. He told to change the string to file type and there would be less hassle.
So i pulled up `irb` and started some experimenting with DataTypes
{% highlight css %}
2.3.0 :001 > str = "i have to test"
2.3.0 :003 > str.class
 => String
2.3.0 :006 > str.is_a?(String)
 => true
2.3.0 :008 > file = File.join('/path', 'name')
 => "/path/name"
2.3.0 :009 > file.class
 => String
2.3.0 :011 > require 'tempfile'
 => true
2.3.0 :013 > tmp = Tempfile.new('foo')
 => #<Tempfile:/var/folders/np/fz6r84d57gg4d8jbnr7z2wd00000gn/T/foo20160514-36498-1chq9a9>
2.3.0 :014 > tmp.class
 => Tempfile
2.3.0 :015 > str.is_a?(File)
 => false
2.3.0 :016 > file_type = StringIO.new(file)
 => #<StringIO:0x007f98d192bdc0>
2.3.0 :017 > file_type.class
 => StringIO
{% endhighlight %}
In some stack overflow post i found that StringIO make string file like, but i tried to access its path
{% highlight css %}
2.3.0 :018 > file_type.path
NoMethodError: undefined method 'path' for #<StringIO:0x007f98d192bdc0>
2.3.0 :019 > tmp.path
 => "/var/folders/np/fz6r84d57gg4d8jbnr7z2wd00000gn/T/foo20160514-36498-1chq9a9"
{% endhighlight %}
Whereas tmp had path method.
{% highlight css %}
2.3.0 :024 > file2 = File.new("/Users/hitenchowdhary/.sequenceserver/164c7c82-c3c0-4522-99db-746a30e3d3da/sequenceserver-xml_report.xml")
 => #<File:/Users/hitenchowdhary/.sequenceserver/164c7c82-c3c0-4522-99db-746a30e3d3da/sequenceserver-xml_report.xml>
2.3.0 :025 > file2.class
 => File
2.3.0 :026 > file2.path
 => "/Users/hitenchowdhary/.sequenceserver/164c7c82-c3c0-4522-99db-746a30e3d3da/sequenceserver-xml_report.xml"
{% endhighlight %}
This was success for File DataType but it had one issue. The `File.new(filename)` was only success when the filename already exists but we had to first create and empty file the populate it.

Its been a while i have tried a few methods to create the file and populate it. But the creation is taking fine, but the `blast_formatter` doesn’t seems to be able to write in the file.

With files this is also a rule to follow: When open a file, write on it but the write won’t actually take place until you close the file. And write will only take place with `File.write()` method. Now here is the catch `blast_formatter` is directly putting the output in `/path/to/file`.
{% highlight css %}
placeholder = "/Users/hitenchowdhary/Documents/Seqserv/temp.txt"
  command =
   "blast_formatter -archive '#{archive_file}'" \
   " -outfmt '#{format} #{specifiers}'" \
   " -out '#{placeholder}' 2> /dev/null"
  logger.debug("Executing: #{command}")
  Dir.chdir(File.exist?(DOTDIR) && DOTDIR || Dir.pwd) do
   system(command)
   data = File.read(placeholder)
   logger.debug("File path: #{xfile.path}")
   File.write(xfile.path, data)
   xfile.close
   logger.debug("Size check: #{File.size(xfile.path)}")
  end
{% endhighlight %}
Here the data dump is working just fine, but no idea why the `File.write` is not working properly.
Some replacement for File.write() worked. `xfile.write` worked but only for tsv not for xml.
