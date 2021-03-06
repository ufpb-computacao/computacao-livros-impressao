require 'net/http'
require 'rake/clean'
require 'find'
require 'date'
require 'open3'
require 'yaml'
require 'pdf-reader'

task :default => ["colecao:build"]

GITHUB_REPO = `git config remote.origin.url`.strip.gsub('git@github.com:','').gsub('.git','')

build = YAML.load_file('config/build.yml')
volumes = YAML.load_file('config/volumes.yml')

TARGET_DIR = 'target'
RELEASE_DIR = 'releases'
CACHE_DIR = 'cache'
VERSION = build['version']
SEJDA = build['sejda']


directory TARGET_DIR
directory CACHE_DIR
directory RELEASE_DIR

CLEAN.include(TARGET_DIR)

namespace "colecao" do

  desc 'Build coleção'
  task :build => [TARGET_DIR, CACHE_DIR]

  desc 'Create release files'
  task :release => [RELEASE_DIR, 'colecao:build']

  desc 'Download all released books'
  multitask 'download'
  
  desc 'Imprime quantidade de páginas'
  task :paginas  
  
  volumes.each do |name,livros|
    target_volume_file = "#{TARGET_DIR}/#{name}-#{VERSION}.pdf"
    release_volume_file = "#{RELEASE_DIR}/#{name}-#{VERSION}.pdf"
    
    desc 'Build volume file'
    file target_volume_file
    
    books_after_stamps = []
    

    livros.each do |livro|
      cache_path = ''
      copy_name = 'undefined'
      if (livro['source']) then
        copy_name = livro['source'].split('/')[-1]
        cache_path = "#{CACHE_DIR}/#{copy_name}"
        file livro['source']
        file cache_path => [livro['source'],CACHE_DIR] do |t|
          cp t.prerequisites.first, t.name
        end
      elsif (livro['url']) then
        copy_name = livro['url'].split('/')[-1]
        cache_path = "#{CACHE_DIR}/#{copy_name}"
        file cache_path => [CACHE_DIR] do |t|
          `wget --output-document=#{t.name} #{livro['url']}`
        end
      elsif (livro['isbn']) then
        next
      end
      copy_path  = "#{TARGET_DIR}/#{copy_name}"
      file copy_path => [cache_path,TARGET_DIR] do |t|
        cp t.prerequisites.first, t.name
      end

      if (livro['stamp']) then
        livro_with_stamp = copy_path.ext('with_stamp.pdf')
        livro_bookmark = copy_path.ext('bookmark')

        file livro_bookmark => [copy_path] do
          puts "Extraindo info de #{copy_path}\n"
          `pdftk #{copy_path} dump_data output #{livro_bookmark}`
          puts "pdftk #{copy_path} dump_data output #{livro_bookmark}"
        end

        #desc 'Apply stamp to the book'
        file livro_with_stamp => [copy_path, livro_bookmark ] do
          livro_tmp = copy_path.ext('tmp.pdf')

          `pdftk #{copy_path} stamp #{livro['stamp']} output #{livro_tmp}`
          `pdftk #{livro_tmp} update_info #{livro_bookmark} output #{livro_with_stamp}`

          rm_rf livro_tmp
        end
        books_after_stamps << livro_with_stamp
        file target_volume_file => [livro_with_stamp, livro_bookmark]
      else
        books_after_stamps << copy_path
        file target_volume_file => [copy_path]
      end
    end

    file target_volume_file do
      source_files = books_after_stamps.join ' '
      `#{SEJDA} merge -f #{source_files} -o #{target_volume_file} --addBlanks`
    end

    file release_volume_file => [target_volume_file] do |t|
      cp t.prerequisites.first, t.name
    end
    
#    desc "imprime o número de páginas de #{release_volume_file}"
    task "#{release_volume_file}_paginas" => release_volume_file do |t|
      reader = PDF::Reader.new(t.prerequisites.first)
      puts "#{t.prerequisites.first}: #{reader.page_count} páginas"
    end
    task "paginas" => "#{release_volume_file}_paginas"


    task :build => target_volume_file
    task :release => release_volume_file

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
