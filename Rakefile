# encoding: UTF-8
require "rubygems"
require "bundler"

Bundler.require

ROOT = File.expand_path(File.dirname(__FILE__))
DOCS = Hash.new {|h,k| h[k] = Nokogiri::HTML(File.open(k), "r")}

namespace :process do
  task :execute => [
    :remove_included_styles,
    :remove_included_scripts,
    :fix_title_in_head,
    :add_link_fixup_script_into_head,
    :remove_on_load_from_body,
    :wrap_header_group,
    :add_main_section,
    :add_wrapper,
    :insert_head,
    :remove_extra_charsets_in_head,
    :references,
    :header,
    :footer,
    :analytics,
    :search_index,
    :insert_search,
    :insert_javascripts,
    :insert_syncing,
    :insert_manifest,
    :add_next_up_links,
    :remove_comments,
    :remove_dom_interface,
    :toc,
    :fancy_numbers_in_toc,
    :remove_back_and_forward_links,
    :remove_bottom_nav_links,
    :transform_index,
    :write_docs,
    :add_generation_time
  ]

  def each_page(&block)
    Dir[File.join(ROOT, 'public', "*.html")].each do |path|
      yield DOCS[path], path
    end
  end

  task :write_docs do
    DOCS.each do |path, doc|
      File.open(path, "w") {|file| file << doc.to_html }
    end
  end

  def insert(markup_path, selector, method = :add_child)
    markup = File.open(markup_path, "r").read

    each_page do |doc, filename|
      element = doc.at(selector)
      if element
        element.send(method, Nokogiri::HTML::fragment(markup))
      else
        puts "No element for selector #{selector} in #{filename}"
      end
    end
  end

  task :remove_included_styles do
    each_page do |doc, filename|
      doc.css("link[rel='stylesheet'], style, link[href^='data:text/css']").remove
    end
  end

  task :remove_included_scripts do
    each_page do |doc, filename|
      doc.css("script").remove
    end
  end

  task :fix_title_in_head do
    each_page do |doc, filename|
      title = doc.css("head title")
      title.first.content = "HTML: The Living Standard - Edition for Web Developers"
    end
  end

  task :add_link_fixup_script_into_head do
    link_fixup = '<script src="link-fixup.js"></script>'
    each_page do |doc, filename|
      last_child = doc.css("head").children.last
      last_child.after(link_fixup)
    end
  end

  task :remove_on_load_from_body do
    each_page do |doc, filename|
      doc.xpath('//@onload').remove
    end
  end

  task :wrap_header_group do
    each_page do |doc, filename|
      # Wrap the header group in an a referencing "/"
      toc = doc.at("h2#table-of-contents")
      if toc
        toc.add_previous_sibling("<h1><a href='/' rel='home'>HTML: The Living Standard</a></h1>")
      end
    end
  end

  task :add_wrapper do
    each_page do |doc, filename|
      body = doc.at("body")
      if body
        body.children = Nokogiri::HTML::fragment("<div class='wrapper'>#{body.to_html}</div>")
      end
    end
  end

  # we'll get everything within the body, wrap it within a <section role="main">
  task :add_main_section do
    each_page do |doc, filename|
      body = doc.at("body")
      if body
        body.children = Nokogiri::HTML::fragment("<section role='main'>#{body.to_html}</section>")
      end
    end
  end

  desc "Does some special transformations on the index.html file"
  task :transform_index do
    doc = DOCS[File.join(ROOT, 'public', 'index.html')]
    main_section = doc.at("section[role='main']")
    if main_section
      main_section.css("ol.toc").after(File.open("html/credits.html", "r").read)
      main_section.css("> h1:first-child").remove
      main_section.css("ol.brief.toc").remove
      main_section.css("#contents").remove
    end

    # Remove hashes from links to top-level pages
    # (This stops the index toc from skipping past the header for top level items)
    toc_links = doc.css("ol.toc li a").to_a
    toc_links.each.with_index do |link, index|
      previous_link = toc_links[index - 1]
      previous_href = previous_link && previous_link.attributes["href"].to_s.gsub(/#.*\Z/, "")

      this_href = link.attributes["href"].to_s.gsub(/#.*\Z/, "")

      if previous_link && previous_href == this_href
        puts "Not removing anchor from #{this_href}, previous is #{previous_href}"
      else
        link.attributes["href"].value = this_href
      end
    end

    # Remove links that are the same page as their parent, remove them.
    #doc.css("ol.toc li a").each do |item|
    #  item_href = item.attributes["href"].to_s.gsub(/#.*\Z/, "")
    #  branch = item.parent.parent.parent

    #  if branch.node_name == "li"
    #    a = branch.at("a")
    #    a_href = a.attributes["href"].to_s.gsub(/#.*\Z/, "")

    #    if a_href == item_href
    #      item.remove
    #    end
    #  end
    #end
  end

  desc "Add document footer"
  task :footer do
    insert("html/footer.html", "body .wrapper")
  end

  desc "Add document header"
  task :header do
    insert("html/header.html", "body .wrapper > *", :before)
  end

  desc "Add analytics"
  task :analytics do
    insert("html/analytics.html", "body")
  end

  desc "Pull references inline"
  task :references do
    reference_doc = Nokogiri::HTML(File.open("public/references.html", "r"))

    each_page do |doc, filename|
      doc.css("a[href*='#refs']").each do |reference_link|
        reference_string = reference_link.attributes["href"].value.gsub(/^(.*)\#/, "")
        aside_content = reference_doc.css("dt#" + reference_string + " + dd").inner_html

        # Create aside element above the parent of the link
        unless doc.css("aside#" + reference_string).any?
          wrapper = reference_link.parent.replace(Nokogiri::HTML::fragment("<div class='reference-wrapper'>#{reference_link.parent.to_s}</div>"))[0]
          wrapper.add_child('<aside id="'+ reference_string +'" class="reference">'+ aside_content +'</aside>')
        end
      end
    end
  end

  desc "Add a search index json file"
  task :search_index do
    fork do
      toc = Nokogiri::HTML(File.open("public/index.html", "r"))
      index = toc.css("ol.toc li a").inject([]) do |index, link|
        section = link.css("span")
        section_text = section.text.strip
        section.remove

        parent_section = link.parent.parent.parent

        if parent_section.node_name == "li"
          section_text = "#{section_text} — #{parent_section.at("a").text}"
        end

        index << {
          :uri => link.attributes["href"],
          :text => link.text.strip,
          :section => section_text
        }
      end

      File.open("public/search_index.json", "w") {|buffer| buffer << JSON.generate(index)}
    end
  end

  desc "Add search to each html file"
  task :insert_search do
    #insert("html/search.html", "header.head")
  end

  desc "Insert javascripts"
  task :insert_javascripts do
    Dir["public/**/*.js"].each do |js_path|
      js_path = js_path.gsub("public/", "")

      each_page do |doc, filename|
        body = doc.at("body")
        if body
          body.add_child('<script src="/' + js_path + '" defer>')
        else
          puts "No body in #{filename}"
        end
      end
    end
  end

  desc "Add manifest reference"
  task :insert_manifest => "generate:manifest" do
    each_page do |doc, filename|
      html = doc.at("html")
      if html
        html['manifest'] = "/offline.manifest"
      else
        puts "No <html> in #{filename}"
      end
    end
  end

  desc "Insert syncing notification"
  task :insert_syncing do
    insert("html/syncing.html", "body")
  end

  task :insert_head do
    head = File.open("html/head.html", "r").read
    each_page do |doc, filename|
      first_child = doc.css("head").children.first
      first_child.before(head)
    end
  end

  task :remove_extra_charsets_in_head do
    Dir[File.join(ROOT, 'public', "*.html")].each do |path|
      charsets = DOCS[path].css("meta[charset]")
      charsets[1].remove if charsets.length > 1
    end
  end

  desc "Add 'next up' page links"
  task :add_next_up_links do
    each_page do |doc, filename|
      next_page = doc.at("link[rel='next']") || doc.at("nav a:last-child")

      unless next_page
        puts "No 'next' link found for #{filename}"
        next
      end

      title = next_page.content.to_s.gsub("→", "")
      href = next_page.attributes["href"]
      doc.at("footer").before('<div id="up-next"><a href="'+href+'"><p>Up next</p><h1>'+title+'</h1></a></div>')
    end
  end

  task :remove_bottom_nav_links do
    each_page do |doc, filename|
      doc.css("section[role='main'] nav:last-child").remove
    end
  end

  task :remove_comments do
    Dir.chdir("public") do
      Dir["*.html"].each do |page|
        without_comments = File.open(page, "r").read.gsub(/<!--(.*)-->/, "")
        File.open(page, "w") {|buffer| buffer << without_comments }
      end
    end
  end

  task :remove_dom_interface do
    each_page do |doc, filename|
      dt = doc.xpath('//dt[text()="DOM interface:"]')
      dt.xpath('following-sibling::dd[1]').remove
      dt.remove
    end
  end

  task :toc do
    each_page do |doc, filename|
      if nav = doc.at("section[role='main'] nav")
        nav.children.first.before("<button id='toc-toggle'>In this section&hellip;</button>") if nav.css("ol").any?
        #doc.css("section[role='main'] nav > a").remove
        nav.replace(Nokogiri::HTML::fragment(nav.to_s.gsub("&ndash;", "")))
      end
    end
  end

  task :fancy_numbers_in_toc do
    each_page do |doc, filename|
      doc.css("ol.toc a").each do |toc_link|
        link_content = toc_link.inner_html
        replaced_link_content = link_content.gsub(/^\d+(.\d+)*/) {|number| "<span class=\"secno\">#{number}</span>"}

        toc_link.inner_html = Nokogiri::HTML::fragment(replaced_link_content)
      end
    end
  end

  task :remove_back_and_forward_links do
    each_page do |doc, filename|
      doc.css("section[role='main'] > nav:first-child").remove
    end
  end

  task :add_generation_time do
    Dir.chdir("public") do
      Dir["*.html"].each do |page|
        File.open(page, "a") {|buffer| buffer << "<!--Regenerated #{Time.now.to_s} -->" }
      end
    end
  end
end

namespace :generate do

  desc "Generate a HTML5 manifest for offline specifications"
  task :manifest do
    files = Dir["public/**/*.*"].map{|fp| fp.gsub("public/", "") }.join("\n")

    MANIFEST = %Q{CACHE MANIFEST
      # #{Time.now.to_s}
    #{ files }

      NETWORK:
      http://www.google-analytics.com/ga.js
      http://images.whatwg.org
      https://images.whatwg.org

    }

    File.open("public/offline.manifest", "w") { |buffer| buffer << MANIFEST }
  end
end
