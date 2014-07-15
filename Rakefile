require 'net/http'
require 'rake/clean'
require 'find'
require 'date'
require 'open3'
require 'yaml'

task :default => ["colecao:build"]

GITHUB_REPO = `git config remote.origin.url`.strip.gsub('git@github.com:','').gsub('.git','')

build = YAML.load_file('config/build.yml')
volumes = YAML.load_file('config/volumes.yml')

TARGET_DIR = build['target']
VERSION = build['version']
SEJDA = build['sejda']
CACHE_DIR = 'cache'

directory TARGET_DIR
directory CACHE_DIR


CLEAN.include(TARGET_DIR)


namespace "colecao" do

  desc 'Build coleção'
  task :build

  volumes.each do |name,livros|
    target_volume_file = "#{TARGET_DIR}/#{name}-#{VERSION}.pdf"
    desc 'Build volume file'
    file target_volume_file
    desc 'Download all released books'
    multitask 'download'

    books_with_stamps = []

    livros.each do |livro|
      cach_path = ''
      copy_path = ''
      if (livro['source']) then
        copy_name = livro['source'].split('/')[-1]
        copy_path = "#{TARGET_DIR}/#{copy_name}"
        cache_path = "#{CHACHE_DIR}/#{copy_name}"
        file livro['source']
        #desc 'copy from source'
        file copy_path => [TARGET_DIR,livro['source']] do
          cp livro['source'], copy_path
        end
        file cache_path => [CACHE_DIR,livro['source']] do
          cp livro['source'], cache_path
        end

      end
      if (livro['url']) then
        copy_name = livro['url'].split('/')[-1]
        copy_path  = "#{TARGET_DIR}/#{copy_name}"
        cache_path = "#{CACHE_DIR}/#{copy_name}"
        file copy_path => [TARGET_DIR] do
          `wget --output-document=#{copy_path} #{livro['url']}`
        end
        file cache_path => [CACHE_DIR] do
          `wget --output-document=#{cache_path} #{livro['url']}`
        end
      end

      livro_with_stamp = copy_path.ext('with_stamp.pdf')
      livro_bookmark = copy_path.ext('bookmark')

      file livro_bookmark => [copy_path, cache_path] do
        `pdftk #{copy_path} dump_info output #{livro_bookmark}`
      end

      #desc 'Apply stamp to the book'
      file livro_with_stamp => [copy_path, livro_bookmark ] do
        livro_tmp = copy_path.ext('tmp.pdf')

        `pdftk #{copy_path} stamp #{livro['stamp']} output #{livro_tmp}`
        `pdftk #{livro_tmp} update_info #{livro_bookmark} output #{livro_with_stamp}`

        rm_rf livro_tmp

      end


      books_with_stamps << livro_with_stamp

      file target_volume_file => [livro_with_stamp, livro_bookmark]
    end

    file target_volume_file do
      source_files = books_with_stamps.join ' '
      `#{SEJDA} merge -f #{source_files} -o #{target_volume_file} --addBlanks`
    end

    task :build => target_volume_file
  end


  task :t do
    volumes.each do |name,livros|
      puts name
    end
  end

  task :stamp do
    livros.each_with_index do |livro, i|
      source = "#{TARGET}/#{i}/livro.pdf"
      target = "#{TARGET}/#{i}/livro_with_stamp.pdf"
      `pdftk #{source} stamp #{livro['stamp']} output #{target}`
    end
  end

  task :merge do

    target = "#{TARGET}/computacao-periodo1.pdf"
    sources = []
    livros.each_with_index do |livro, i|
      sources << "#{TARGET}/#{i}/livro_with_stamp.pdf"
    end

    source_files = sources.join ' '
    `#{SEJDA} merge -f #{source_files} -o #{target}`
  end
end









namespace "tag" do

  desc "List project tags"
  task :list do
    sh "git tag --list"
  end

  desc "Aplly a tag to the project. The tag can be used as the edition."
  task :apply, [:tag] do |t, args|
    sh "git status"
    sh "git tag -a #{args.tag} -m 'Gerando versão #{args.tag}'"
  end

  desc "Delete a tag applied."
  task :delete, [:tag] do |t,args|
    sh "git tag -d #{args.tag}"
  end

  desc "Push tags"
  task "push" do
    sh "git push origin --tags"
  end

end

#desc "Download new Rakefile"
#task :uprake do
#  `wget --output-document=Rakefile https://raw.githubusercontent.com/edusantana/novo-livro/master/Rakefile`
#end

namespace "github" do
  desc "List issues from github milestone"
  task :issues, [:milestone] do |t,args|
    puts "Acessing: #{GITHUB_REPO} milestone=#{args.milestone}"
    require 'octokit'
#    require 'highline/import'
    client = Octokit::Client.new
    milestone = nil
    milestones = client.list_milestones(GITHUB_REPO)
    opcoes = milestones.map {|m| m[:title]}

    if (args.milestone) then
      milestones.each do |m|
        if m[:title] == args.milestone then
          milestone = m
        end
      end
    else
      milestone = milestones[-1]
    end
    puts "Milestone: #{milestone[:title]}"

    issues = client.list_issues(GITHUB_REPO, state:'Closed', milestone:milestone[:number], direction:'asc')
    issues.each do |i|
      puts "- #{i[:title]} (##{i[:number]})."
    end
  end
end
