require 'aws-sdk'
require 'rake'
require 'uri'
require 'os'
require 'rake/clean'
require 'shellwords'
require 'nokogiri'
require 'openssl'
require 'net/http'
require 'json'
require 'fileutils'
require 'typhoeus'
require 'time'
require 'date'
require 'pp'
require 'zip'
require 'tempfile'
require 'rubygems/package'
require 'zlib'
require 'open3'
require 'yaml'
require 'washbullet'
require 'rake/loaders/makefile'
require 'selenium-webdriver'
require 'rchardet'
require 'csv'
require 'base64'
require 'net/http/post/multipart'
require 'facets'
require 'rest-client'
require_relative 'lib/unicode_table'

TIMESTAMP = DateTime.now.strftime('%Y-%m-%d %H:%M:%S')

def cleanly(f)
  begin
    yield
  rescue
    FileUtils.rm_f(f)
    raise
  end
end

#ABBREVS = YAML.load_file('resource/abbreviations/lists.yml')
#ABBREVS.each{|a|
#  if File.basename(a['path']) == 'WOS.json'
#    file a['path'] => 'Rakefile' do |t|
#      tmp = "tmp/WOS.html"
#      abbrevs = {}
#      (('A'..'Z').to_a + ['0-9']).each{|list|
#        ZotPlus::RakeHelper.download(a['url'].sub('<>', list), tmp)
#        doc = Nokogiri::HTML(open(tmp))
#
#        dt = nil
#        doc.xpath("//*[name()='dt' or name()='dd']").each{|node|
#          next unless %w{dt dd}.include?(node.name)
#          node = node.dup
#          node.children.each{|c| c.unlink unless c.name == 'text'}
#          text = node.inner_text.strip
#          if node.name == 'dt'
#            dt = (text == '' ? nil : text.downcase)
#          elsif dt
#            abbrevs[dt] = text
#          end
#        }
#      }
#      abbrevs = { default: { 'container-title' => abbrevs } }
#      open(t.name, 'w'){|f| f.write(JSON.pretty_generate(abbrevs)) }
#    end
#  else
#    file a['path'] => 'Rakefile' do |t|
#      tmp = "tmp/#{File.basename(t.name)}"
#      ZotPlus::RakeHelper.download(a['url'], tmp)
#
#      text = open(tmp).read
#      cd = CharDet.detect(text)
#      if !%w{ascii utf-8}.include?(cd['encoding'].downcase)
#        text = text.encode('utf-8', cd['encoding'])
#        open(tmp, 'w'){|f| f.write(text) }
#      end
#
#      abbrevs = {}
#      IO.readlines(tmp).each{|line|
#        line.strip!
#        next if line[0] == '#'
#        next unless line =~ /=/
#        line = line.split('=', 2).collect{|t| t.strip}
#        next if line.length != 2
#        journal, abbrev = *line
#        journal.downcase!
#        abbrev.sub(/\s*;.*/, '')
#        next if journal == '' || abbrev == ''
#
#        terms = journal.split(/(\[[^\]]+\])/).reject{|t| t.strip == ''}.collect{|t|
#          if t[0] == '[' && t[-1] == ']'
#            OpenStruct.new(term: t[1..-2].downcase)
#          else
#            OpenStruct.new(term: t.downcase, required: true)
#          end
#        }
#
#        (1..2**terms.length).collect{|permutation|
#          (0..terms.length-1).collect{|term|
#            terms[term].required || ((1 << term) & permutation) != 0 ?  terms[term].term : nil
#          }.compact.join(' ').strip
#        }.collect{|name|
#          name.gsub!(/[^a-z]+/, ' ')
#          name.gsub!(/\s+/, ' ')
#          name.strip
#        }.uniq.each{|journal|
#          abbrevs[journal] = abbrev
#        }
#      }
#      abbrevs = { default: { 'container-title' => abbrevs } }
#      open(t.name, 'w'){|f| f.write(JSON.pretty_generate(abbrevs)) }
#    end
#  end
#}
ZIPFILES = (Dir['{chrome,resource}/**/*.{coffee,pegjs}'].collect{|src|
  tgt = src.sub(/\.[^\.]+$/, '.js')
  tgt
}.flatten + Dir['chrome/**/*.xul'] + Dir['chrome/{skin,locale}/**/*.*'] + Dir['resource/translators/*.yml'].collect{|tr|
  root = File.dirname(tr)
  stem = File.basename(tr, File.extname(tr))
  %w{header.js js json}.collect{|ext| "#{root}/#{stem}.#{ext}" }
}.flatten + [
  'chrome/content/zotero-better-bibtex/fold-to-ascii.js',
  'chrome/content/zotero-better-bibtex/punycode.js',
  'chrome/content/zotero-better-bibtex/lokijs.js',
  'chrome/content/zotero-better-bibtex/release.js',
  'chrome/content/zotero-better-bibtex/csl-localedata.js',
  'defaults/preferences/defaults.js',
  'resource/citeproc.js',
  'chrome.manifest',
  'install.rdf',
  'resource/translators/json5.js',
  'resource/translators/preferences.js',
  'resource/translators/latex_unicode_mapping.js',
  'resource/translators/xregexp-all.js',
  'resource/reports/cacheActivity.txt',
]).sort.uniq
File.unlink('chrome/content/zotero-better-bibtex/release.js') if File.file?('chrome/content/zotero-better-bibtex/release.js')

CLEAN.include('{resource,chrome,defaults}/**/*.js')
CLEAN.include('{resource,chrome,defaults}/**/*.js.map')
CLEAN.include('tmp/**/*')
CLEAN.include('resource/translators/*.json')
CLEAN.include('.depend.mf')
CLEAN.include('*.xpi')
CLEAN.include('*.log')
CLEAN.include('*.cache')
CLEAN.include('*.debug')
CLEAN.include('*.dbg')
CLEAN.include('*.tmp')

FileUtils.mkdir_p 'tmp'

class String
  def shellescape
    Shellwords.escape(self)
  end
end

require 'zotplus-rakehelper'

def saveAbbrevs(abbrevs, file, jurisdiction='default')
  abbrevs.keys.each{|journal|
    if journal == abbrevs[journal]
      abbrevs.delete(journal)
#    else
#      dc = journal.downcase.gsub(/[^a-z]+/, ' ').gsub(/\s+/, ' ').strip # maybe not...
#      if dc != journal
#        abbrevs[dc] = abbrevs[journal]
#        abbrevs.delete(journal)
#      end
    end
  }
  json = {}
  json[jurisdiction] = { 'container-title' => abbrevs }
  open(file, 'w'){|f| f.write(JSON.pretty_generate(json)) }
end

DOWNLOADS = {
  'chrome/content/zotero-better-bibtex' => {
    'test/chai.js'      => 'http://chaijs.com/chai.js',
    'test/yadda.js'     => 'https://raw.githubusercontent.com/acuminous/yadda/master/dist/yadda-0.11.5.js',
    'lokijs.js'         => 'https://raw.githubusercontent.com/techfort/LokiJS/master/build/lokijs.min.js',
  },
  'resource/translators' => {
    'unicode.xml'         => 'http://www.w3.org/Math/characters/unicode.xml',
    'org.js'              => 'https://raw.githubusercontent.com/mooz/org-js/master/org.js',
  },
}
DOWNLOADS.each_pair{|dir, files|
  files.each_pair{|file, url|
    file "#{dir}/#{file}" => 'Rakefile' do |t|
      ZotPlus::RakeHelper.download(url, t.name)
    end
  }
}

file 'defaults/preferences/defaults.js' => ['defaults/preferences/defaults.yml', 'Rakefile'] do |t|
  prefs = YAML::load_file(t.source)
  open(t.name, 'w'){|f|
    prefs.each_pair{|k, v|
      k = "extensions.zotero.translators.better-bibtex.#{k}"
      f.puts("pref(#{k.to_json}, #{v.to_json});")
    }
  }
end

file 'resource/translators/preferences.js' => ['defaults/preferences/defaults.yml', 'Rakefile'] do |t|
  open(t.name, 'w'){|f|
    f.puts("Translator.preferences = #{JSON.pretty_generate(YAML::load_file(t.source))}")
  }
end

# someone thinks HTML-loaded javascripts are harmful. If that were true, you have bigger problems than this
# people.
file 'resource/reports/cacheActivity.txt' => 'resource/reports/cacheActivity.html' do |t|
  FileUtils.cp(t.source, t.name)
end

file 'resource/citeproc.js' => 'Rakefile' do |t|
  cleanly(t.name) do
    ZotPlus::RakeHelper.download('https://bitbucket.org/fbennett/citeproc-js/raw/tip/citeproc.js', t.name)
    sh "#{NODEBIN}/grasp -i -e 'thedate[DATE_PARTS_ALL[i]]' --replace 'thedate[CSL.DATE_PARTS_ALL[i]]' #{t.name.shellescape}"
    sh "#{NODEBIN}/grasp -i -e 'if (!Array.indexOf) { _$ }' --replace '' #{t.name.shellescape}"
    File.rewrite(t.name){|src|
      patched = StringIO.new(src).readlines.collect{|line|
        if line.strip == 'if (!m1split[i-1].match(/[:\\?\\!]\\s*$/)) {'
          line.sub(/if.*{/, 'if (i > 0 && !m1split[i-1].match(/[:\\?\\!]\\s*$/)) {')
        else
          line
        end
      }.join('')
      open('https://raw.githubusercontent.com/zotero/zotero/4.0/chrome/content/zotero/xpcom/citeproc-prereqs.js').read + patched + "\nvar EXPORTED_SYMBOLS = ['CSL'];\n"
    }
    sh "#{NODEBIN}/grasp -i -e 'xmldata.open($a, $b, $c);' --replace 'xmldata.dontopen({{a}}, {{b}}, {{c}});' #{t.name.shellescape}"
    sh "#{NODEBIN}/grasp -i -e 'doc.createElement' --replace 'doc.dontcreateElement' #{t.name.shellescape}"
  end
end

file 'resource/translators/xregexp-all.js' => 'Rakefile' do |t|
  cleanly(t.name) do
    ZotPlus::RakeHelper.download('http://cdnjs.cloudflare.com/ajax/libs/xregexp/2.0.0/xregexp-all.js', t.name)
    # strip out setNatives because someone doesn't like it
    sh "#{NODEBIN}/grasp -i 'func-dec! #setNatives' --replace 'function setNatives(on) { if (on) { throw new Error(\"setNatives not supported in Firefox extension\"); } }' #{t.name}"
  end
end

file 'resource/translators/json5.js' => 'Rakefile' do |t|
  ZotPlus::RakeHelper.download('https://raw.githubusercontent.com/aseemk/json5/master/lib/json5.js', t.name)
end

file 'chrome/content/zotero-better-bibtex/test/tests.js' => ['Rakefile'] + Dir['resource/tests/*.feature'] do |t|
  features = t.sources.collect{|f| f.split('/')}.select{|f| f[0] == 'resource'}.collect{|f|
    f[0] = 'resource://zotero-better-bibtex'
    f.join('/')
  }
  open(t.name, 'w'){|f|
    f.write("Zotero.BetterBibTeX.Test.features = #{features.to_json};")
  }
end

file 'resource/abbreviations/CAplus.json' => 'Rakefile' do |t|
  abbrevs = {}
  doc = Nokogiri::HTML(open('http://www.cas.org/content/references/corejournals'))
  doc.at('//table').xpath('.//tr').each_with_index{|abb, i|
    next if i == 0
    abb = abb.xpath('.//td').collect{|t| t.inner_text.strip}
    next unless abb.length > 1
    next if abb[0] == '' || abb[1] == ''
    abbrevs[abb[0]] = abb[1]
  }
  saveAbbrevs(abbrevs, t.name)
end

file 'resource/abbreviations/J_Entrez.json' => 'Rakefile' do |t|
  record = {}
  abbrevs = {}
  IO.readlines(open('ftp://ftp.ncbi.nih.gov/pubmed/J_Entrez.txt')).each{|line|
    line.strip!
    if line =~ /^-+$/
      abbrevs[record['JournalTitle']] = record['MedAbbr'] if record['JournalTitle'] && record['MedAbbr']
      abbrevs[record['JournalTitle']] = record['IsoAbbr'] if record['JournalTitle'] && record['IsoAbbr']
      record = {}
      next
    end
    line = line.split(':', 2).collect{|t| t.strip}
    next if line.length != 2
    next if line[0] == '' || line[1] == ''
    record[line[0]] = line[1]
  }
  abbrevs[record['JournalTitle']] = record['MedAbbr'] if record['JournalTitle'] && record['MedAbbr']
  abbrevs[record['JournalTitle']] = record['IsoAbbr'] if record['JournalTitle'] && record['IsoAbbr']
  saveAbbrevs(abbrevs, t.name)
end

file 'resource/abbreviations/Science_and_Engineering.json' => 'Rakefile' do |t|
  abbrevs = {}
  jsonp = open('http://journal-abbreviations.library.ubc.ca/dump.php').read
  jsonp.sub!(/^\(/, '')
  jsonp.sub!(/\);/, '')
  doc = Nokogiri::HTML(JSON.parse(jsonp)['html'])
  doc.at('//table').xpath('.//tr').each{|tr|
    tds = tr.xpath('.//td')
    next if tds.length != 2
    journal, abbr = *(tds.collect{|t| t.inner_text.strip})
    next if journal == '' || abbr == ''
    abbrevs[journal] = abbr
  }
  saveAbbrevs(abbrevs, t.name)
end

file 'resource/abbreviations/BioScience.json' => 'Rakefile' do |t|
  abbrevs = {}
  %w{a-b c-g h-j k-q r-z}.each{|list|
    doc = Nokogiri::HTML(open("http://guides.lib.berkeley.edu/bioscience-journal-abbreviations/#{list}"))
    doc.xpath('//table').each{|table|
      table.xpath('.//tr').each_with_index{|tr, i|
        next if i == 0
        tds = tr.xpath('.//td')
        next if tds.length != 2
        journal, abbr = *(tds.collect{|t| t.inner_text.strip})
        next if journal == '' || abbr == ''
        abbrevs[journal] = abbr
      }
    }
  }
  saveAbbrevs(abbrevs, t.name)
  abbrevs = { default: { 'container-title' => abbrevs } }
end

rule '.json' => '.yml' do |t|
  open(t.name, 'w'){|f|
    header = YAML::load_file(t.source)
    header['lastUpdated'] = TIMESTAMP if t.source =~ /\/translators\//

    %w{translatorID label priority}.each{|field|
      raise "Missing #{field} in #{t.source}" unless header[field]
    }

    if !header['translatorType'] || ((header['translatorType'] & 1) == 0 && (header['translatorType'] & 2) == 0) # not import or export
      raise "Invalid translator type #{header['translatorType']} in #{t.source}"
    end

    f.write(JSON.pretty_generate(header))
  }
end

def browserify(code, target)
  puts "\nbrowserify #{target}"
  prefix, code = code.split("\n", 2)
  code, prefix = prefix, code unless code
  prefix = prefix ? prefix + "\n" : ''
  Tempfile.create(['browserify', '.js'], File.dirname(target)) do |caller|
    Tempfile.create(['browserify', '.js'], File.dirname(target)) do |output|
      open(caller.path, 'w'){|f| f.puts code }
      sh "#{NODEBIN}/browserify #{caller.path.shellescape} > #{output.path.shellescape}"
      open(target, 'w') {|f|
        f.write(prefix)
        f.write(open(output.path).read)
      }
    end
  end
end

def utf16literal(str)
  str = str.split(//).collect{|c|
    o = c.ord
    if o >= 0x20 && o <= 0x7E
      c
    elsif o > 0xFFFF
      h = ((o - 0x10000) / 0x400).floor + 0xD800
      l - ((o - 0x10000) % 0x400) + oxDC00
      "\\u#{h.to_s(16).rjust(4, '0')}\\u#{l.to_s(16).rjust(4, '0')}"
    else
      "\\u#{o.to_s(16).rjust(4, '0')}"
    end
  }.join('')
  return "'" + str + "'"
end
file 'chrome/content/zotero-better-bibtex/csl-localedata.coffee' => ['Rakefile'] + Dir['csl-locales/*.xml'] + Dir['csl-locales/*.json'] do |t|
  open(t.name, 'w'){|f|
    f.puts('Zotero.BetterBibTeX.Locales = { months: {}, dateorder: {}}')

    locales = JSON.parse(open('csl-locales/locales.json').read)
    locales['primary-dialects']['en'] = 'en-GB'
    short = locales['primary-dialects'].invert

    locales['language-names'].keys.sort.each{|full|
      names = ["'#{full.downcase}'"]
      names << "'#{short[full].downcase}'" if short[full]
      if full == 'en-US'
        names << "'american'"
      else
        locales['language-names'][full].each{|name|
          names << utf16literal(name.downcase)
          names << utf16literal(name.sub(/\s\(.*/, '').downcase)
        }
      end
      names.uniq!
      names = names.collect{|name| "Zotero.BetterBibTeX.Locales.dateorder[#{name}]" }.join(' = ')

      locale = Nokogiri::XML(open("csl-locales/locales-#{full}.xml"))
      locale.remove_namespaces!
      order = locale.xpath('//date[@form="numeric"]/date-part').collect{|d| d['name'][0]}.join('')
      f.puts("#{names} = #{order.inspect}")

      months = 1.upto(12).collect{|month| locale.at("//term[@name='month-#{month.to_s.rjust(2, '0')}' and not(@form)]").inner_text.downcase }
      seasons = 1.upto(4).collect{|season| locale.at("//term[@name='season-#{season.to_s.rjust(2, '0')}']").inner_text.downcase }

      months = '[' + (months + seasons).collect{|name| utf16literal(name) }.join(', ') + ']'

      f.puts("Zotero.BetterBibTeX.Locales.months[#{full.inspect}] = #{months}")
    }
  }
end

file 'resource/translators/marked.js' => 'Rakefile' do |t|
  browserify("var LaTeX; if (!LaTeX) { LaTeX = {}; };\nLaTeX.marked=require('marked');", t.name)
end

file 'resource/translators/acorn.js' => 'Rakefile' do |t|
  browserify("acorn = require('../../node_modules/acorn/dist/acorn_csp');", t.name)
end

file 'chrome/content/zotero-better-bibtex/fold-to-ascii.js' => 'Rakefile' do |t|
  browserify("Zotero.BetterBibTeX.removeDiacritics = require('fold-to-ascii').fold;", t.name)
end

file 'chrome/content/zotero-better-bibtex/punycode.js' => 'Rakefile' do |t|
  browserify("Zotero.BetterBibTeX.punycode = require('punycode');", t.name)
end

file 'chrome/content/zotero-better-bibtex/release.js' => 'install.rdf' do |t|
  open(t.name, 'w') {|f| f.write("
      Zotero.BetterBibTeX.release = #{RELEASE.to_json};
    ")
  }
end

rule( /\.header\.js$/ => [ proc {|task_name| [task_name.sub(/\.header\.js$/, '.yml'), 'Rakefile', 'install.rdf'] } ]) do |t|
  header = YAML.load_file(t.source)
  open(t.name, 'w'){|f|
    f.write("
      Translator.header = #{header.to_json};
      Translator.release = #{RELEASE.to_json};
      Translator.#{header['label'].gsub(/[^a-z]/i, '')} = true;
    ")
  }
end

task :amo => XPI do
  amo = XPI.sub(/\.xpi$/, '-amo.xpi')

  Zip::File.open(amo, 'w') do |tgt|
    Zip::File.open(XPI) do |src|
      src.each do |entry|
        data = entry.get_input_stream.read
        if entry.name == 'install.rdf'
          data = Nokogiri::XML(data)
          data.at('//em:updateURL').unlink
          data = data.to_xml
        end
        tgt.get_output_stream(entry.name) { |os| os.write data }
      end
    end
  end
end

task :test, [:tag] => [XPI, :plugins] + Dir['test/fixtures/*/*.coffee'].collect{|js| js.sub(/\.coffee$/, '.js')} do |t, args|
  tag = ''

  if args[:tag] =~ /ci-cluster-(.*)/
    clusters = 4
    cluster = $1
    if cluster == '*'
      tag = "--tag ~@noci"
    elsif cluster =~ /^[0-9]$/ && cluster.to_i < (clusters - 1)
      tag = "--tag ~@noci --tag @test-cluster-#{cluster}"
    else
      tag = "--tag ~@noci " + (0..clusters - 2).collect{|n| "--tag ~@test-cluster-#{n}" }.join(' ')
    end

  else
    tag = "@#{args[:tag]}".sub(/^@@/, '@')

    if tag == '@'
      tag = "--tag ~@noci"
    elsif tag == '@all'
      tag = ''
    else
      tag = "--tag #{tag}"
    end
  end

  puts "Tests running: #{tag}"


  cucumber = "cucumber --require features --strict #{tag} resource/tests"
  if ENV['CI'] == 'true'
    sh cucumber
  else
    begin
      if OS.mac?
        sh "script -q -t 1 cucumber.run #{cucumber}"
      else
        sh "script -ec '#{cucumber}' cucumber.run"
      end
    ensure
      sh "sed -re 's/\\x1b[^m]*m//g' cucumber.run | col -b > cucumber.log"
      sh "rm -f cucumber.run"
    end
  end
end

task :debug => XPI do
  xpi = Dir['*.xpi'][0]
  dxpi = xpi.sub(/\.xpi$/, '-' + (0...8).map { (65 + rand(26)).chr }.join + '.xpi')
  FileUtils.mv(xpi, dxpi)
  puts dxpi
end

task :share => XPI do |t|
  raise "I can only share debug builds" unless ENV['DEBUGBUILD'] == "true"

  url = URI.parse('http://tempsend.com/send')
  File.open(t.source) do |data|
    req = Net::HTTP::Post::Multipart.new(url.path,
      'file' => UploadIO.new(data, 'application/x-xpinstall', File.basename(XPI)),
      'expire' => '604800'
    )
    res = Net::HTTP.start(url.host, url.port) do |http|
      http.request(req)
    end
    puts "http://tempsend.com#{res['location']}"
  end
end

file 'resource/translators/latex_unicode_mapping.coffee' => 'lib/unicode_table.rb' do |t|
  cleanly(t.name) do
    UnicodeConverter.new.save(t.name)
  end
end

task :markfailing do
  tests = false

  failing = {}
  Dir['features/*.feature'].each{|f| failing[f] = [] }
  IO.readlines('cucumber.log').each{|line|
    tests ||= line =~ /^Failing Scenarios:/
    next unless tests
    next unless line =~ /^cucumber /

    line.sub!(/^cucumber /, '')
    line.sub!(/\s?#.*/, '')
    line.strip!
    file, line = *line.split(':', 2)
    line = Integer(line)
    failing[file] << line
  }

  tags = {}
  failures = 0
  failing.each_pair{|file, lines|
    script = ''
    IO.readlines(file).each_with_index{|line, i|
      lineno = i + 1
      throw "untagged #{file}@#{lineno}: #{line}" if lines.include?(lineno) && !tags[lineno - 1]

      if line !~ /^@/ && line.strip != ''
        script += line
        next
      end

      tags[lineno] = line

      line.gsub!(/@failing[^\s]*/, '')
      line.sub!(/\s+/, ' ')
      line.strip!
      if lines.include?(lineno + 1)
        failures += 1
        line = "@failing @failing-#{failures} #{line}"
      end
      script += line + "\n"
    }

    open(file, 'w'){|f| f.write(script) }
  }
end

task :jasmine do

  profiles = File.expand_path('test/fixtures/profiles/')
  FileUtils.mkdir_p(profiles)
  profile_dir = File.join(profiles, 'default')
  profile = Selenium::WebDriver::Firefox::Profile.new(profile_dir)

  (Dir['*.xpi'] + Dir['test/fixtures/plugins/*.xpi']).each{|xpi|
    profile.add_extension(xpi)
  }

  profile['extensions.zotero.showIn'] = 2
  profile['extensions.zotero.httpServer.enabled'] = true
  profile['dom.max_chrome_script_run_time'] = 6000
  profile['extensions.zotfile.useZoteroToRename'] = true
  profile['browser.download.dir'] = "/tmp/webdriver-downloads"
  profile['browser.download.folderList'] = 2
  profile['browser.helperApps.neverAsk.saveToDisk'] = "application/pdf"
  profile['pdfjs.disabled'] = true
  driver = Selenium::WebDriver.for :firefox, :profile => profile

  driver.navigate.to('chrome://zotero/content/tab.xul')
  output = driver.execute_script('return Object.keys(Zotero);')
  #output = driver.execute_script('return consoleReporter.getLogsAsString();')
  driver.quit

  print output

  # Make sure to exit with code > 0 if there is a test failure
  #raise RuntimeError, 'Failure' unless status === 'success'
end

task :logs, [:id] do |t, args|
  if args[:id].to_s !~ /^[A-Z0-9]+-[A-Z0-9]+$/
    sh "aws s3 ls s3://zotplus-964ec2b7-379e-49a4-9c8a-edcb20db343f"
  else
    sh "aws s3 cp s3://zotplus-964ec2b7-379e-49a4-9c8a-edcb20db343f/#{args[:id]}-errorlog.txt tmp/#{args[:id]}-errorlog.txt"
    open("tmp/#{args[:id]}-errorlog-trimmed.txt", 'w'){|trimmed|
      skipnext = false
      IO.readlines("tmp/#{args[:id]}-errorlog.txt").each{|line|
        if line =~ /^\(5\)/
          skipnext = true
        else
          trimmed.write(line) unless skipnext && line.strip == ''
          skipnext = false
        end
      }
    }
    begin
      sh "aws s3 cp s3://zotplus-964ec2b7-379e-49a4-9c8a-edcb20db343f/#{args[:id]}-references.json tmp/#{args[:id]}-references.json"
    rescue
    end
  end
end

task :csltests do
  source = 'test/fixtures/export/(non-)dropping particle handling #313.json'
  testcase = JSON.parse(open(source).read)
  testcase['items'] = testcase['items'].reject{|item| (item['tags'] || []).include?('imported') }

  root = 'https://bitbucket.org/bdarcus/citeproc-test/src/tip/processor-tests/humans/'
  seen = testcase['items'].first['creators'].dup
  Tempfile.create('tests') do |tmp|
    ZotPlus::RakeHelper.download(root, tmp.path)
    tests = Nokogiri::HTML(open(tmp.path))
    n = tests.css('td.filename a').length
    tests.css('td.filename a').each_with_index{|test, i|
      link = URI.join(root, test['href']).to_s.sub(/\?.*/, '').sub('/src/', '/raw/')

      if open(link).read =~ />>=+ INPUT =+>>(.*)<<=+ INPUT =+<</im
        test = JSON.parse($1)
        creators = []
        test.each{|ref|
          %w{author editor container-author composer director interviewer recipient reviewed-author collection-editor %translator}.each{|creator|
            creators << ref[creator] || []
          }
        }
        creators.flatten!
        creators.compact!
        creators = creators.select{|creator|
          (creator.keys & %w{literal dropping-particle non-dropping-particle suffix}).length != 0 || creator['family'] =~ /[^\p{Alnum}]/i
        }.collect{|creator|
          if creator['literal']
            cr = {
              lastName: creator['literal'],
              fieldMode: 1
            }
          else
            cr = {
              creatorType: 'author',
              lastName: [creator['dropping-particle'], creator['non-dropping-particle'], creator['family']].compact.join(' ').strip,
              firstName: [creator['given'], creator['suffix']].compact.join(', '),
            }
            cr.delete(:firstName) if cr[:firstName] == ''
            throw link if cr[:firstName] && creator['isInstitution'] == 'true'
            cr[:fieldMode] = 1 if creator['isInstitution'] == 'true'
          end

          cr[:firstName] =  "François Hédelin, abbé d'" if cr[:firstName] =~ /^François Hédelin/

          cr
        }
        creators = creators.uniq - seen

        seen << creators
        seen.flatten!
        seen.uniq!
      else
        creators = []
      end

      if creators.length > 0
        citekey = link.sub(/.*\//, '').sub(/\..*/, '')
        testcase['items'] << {
          itemType: 'journalArticle',
          title: citekey,
          creators: creators,
          extra: "bibtex: #{citekey}",
          tags: ['imported']
        }
      end
      puts "#{i+1}/#{n}: #{creators.length} creators, #{testcase['items'].length} references"
    }
    open(source, 'w'){|f| f.write(JSON.pretty_generate(testcase)) }
  end
end

#file 'www/better-bibtex/scripting.md' => ['resource/translators/reference.coffee'] do |t|
file 'wiki/Scripting.md' => ['resource/translators/reference.coffee'] do |t|
  sh "markdox --output #{t.name.shellescape} #{t.sources.collect{|s| s.shellescape}.join(' ')}"
  #index = open(t.name).read
  #index = "---\ntitle: Scripting\n---\n" + index
  #open(t.name, 'w'){|f| f.write(index) }
end

task :install => XPI do
  if OS.mac?
    sh "open #{XPI}"
  else
    sh "xdg-open #{XPI}"
  end
end

task :logs2s3 do
  logs = Dir['*.debug'] + Dir['*.log']
  logs = [] if ENV['CI'] == 'true' && ENV['LOGS2S3'] != 'true'

  if (ENV['TRAVIS_PULL_REQUEST'] || 'false') != 'false'
    puts "Logs 2 S3: Not logging pull requests"
  elsif logs.size == 0
    puts "Logs 2 S3: Nothing to do"
  else
    prefix = [ENV['TRAVIS_BRANCH'], ENV['TRAVIS_JOB_NUMBER']].select{|x| x}.join('-')
    prefix += '-' if prefix != ''

    form = JSON.parse(open('https://zotplus.github.io/s3.json').read)
    url = URI.parse(form['action'])
    path = url.path
    path = '/' if path == ''
    params = form['fields']

    logs.each{|log|
      puts "Logs 2 S3: #{log}"
      params[form['filefield']] = UploadIO.new(File.expand_path(log), 'text/plain', "#{prefix}#{log}")
      req = Net::HTTP::Post::Multipart.new(path, params)
      http = Net::HTTP.new(url.host, url.port)
      http.use_ssl = true if url.scheme == 'https'
      res = http.start do |http|
        http.request(req)
      end
    }
  end
end

task :xpi do
  puts XPI
end

