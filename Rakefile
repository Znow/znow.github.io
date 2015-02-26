require 'rake'
require 'yaml'
require 'time'

desc 'create a new draft post'
task :post do
    title = ENV['title']
    slug = "#{Date.today}-#{title.downcase.gsub(/[^\w]+/, '-')}" if title

    file = File.join(
        File.dirname(__FILE__),
        '_posts',
        slug + '.markdown'
    )

    File.open(file, "w") do |f|
        f << <<-EOS.gsub(/^     /, '')
        ---
        layout: post
        title: #{title}
        published: false
        categories:
        ---

        EOS
    end

    system ("#{ENV['EDITOR']} #{file}")
end

desc 'List all draft posts'
task :drafts do
  puts `find ./_posts -type f -exec grep -H 'published: false' {} \\;`
end