require 'rubygems'
require 'bundler/setup'
Bundler.require(:default)
require 'net/http'
require 'uri'
require 'messaging'

# read in and evaluate an external settings file
eval File.open('settings.rb').read if File.exists?('settings.rb')

nakamura = [{"path" => "../sparsemapcontent", "repository" => "https://github.com/ieb/sparsemapcontent.git"},
  {"path" => "../solr", "repository" => "https://github.com/ieb/solr.git"},
  {"path" => "../nakamura", "remote" => "sakaiproject", "repository" => "https://github.com/sakaiproject/nakamura.git"}] if nakamura.nil?

ui = {"path" => "../3akai-ux", "repository" => "https://github.com/sakaiproject/3akai-ux.git"} if ui.nil?

num_users_groups = 5

# setup java command and options
JAVA_EXEC = "java" if !defined? JAVA_EXEC
JAVA_OPTS = "-Xms256m -Xmx1024m -XX:PermSize=64m -XX:MaxPermSize=512m" if !defined? JAVA_OPTS
if !defined? JAVA_DEBUG_OPTS then
  if defined? JAVA_DEBUG and JAVA_DEBUG then
    JAVA_DEBUG_OPTS = "-Xdebug -Xrunjdwp:transport=dt_socket,address=8500,server=y,suspend=n"
  else
    JAVA_DEBUG_OPTS = ""
  end
end
APP_OPTS = "" if !defined? APP_OPTS
JAVA_CMD = "#{JAVA_EXEC} #{JAVA_OPTS} #{JAVA_DEBUG_OPTS}" if !defined? JAVA_CMD

# setup maven command and options
MVN_EXEC = "mvn" if !defined? MVN_EXEC
MVN_OPTS = "-Dmaven.test.skip" if !defined? MVN_OPTS
MVN_CMD = "#{MVN_EXEC} #{MVN_OPTS}" if !defined? MVN_CMD

CLEAN_FILES = ["./derby.log", "./sling", "./activemq-data", "./store"]

puts "Using settings:"
puts "JAVA: #{JAVA_CMD}"
puts "MVN:  #{MVN_CMD}"
p ui
p nakamura
puts ""

# include external rake file for custom tasks
Dir.glob('*.rake').each { |r| import r }

########################
##  Task Definitions  ##
########################
desc "Clone the repositories needed to build everything."
task :clone do
  cmds = []
  if ui.has_key? "path"
    if File.directory? ui["path"]
      puts "#{ui["path"]} already exists."
    elsif ui.has_key? "repository"
      puts "Cloning #{ui["repository"]} to #{ui["path"]}"
      Git.clone(ui["repository"], ui["path"])
      if ui.has_key? "remote" and ui["remote"] != "origin"
        cmds << "(cd #{ui["path"]} && git remote rename origin #{ui["remote"]})"
      end
    end
  end

  for p in nakamura
    if p.has_key? "path"
      if File.directory? p["path"]
        puts "#{p["path"]} already exists."
      elsif p.has_key? "repository"
        puts "Cloning #{p["repository"]} to #{p["path"]}"
        Git.clone(p["repository"], p["path"])
        if p.has_key? "remote" and ui["remote"] != "origin"
          cmds << "(cd #{p["path"]} && git remote rename origin #{p["remote"]})"
        end
      end
    end
  end

  if !cmds.empty?
    puts "\nPlease issue the following commands:"
    cmds.each do |cmd|
      puts cmd
    end
    puts ""
  end
end

desc "Clean files and directories from a previous server start."
task :clean => [:kill] do
  touch CLEAN_FILES
  rm_r CLEAN_FILES
end

desc "Clean the build artifacts generated by the ui build."
task :cleanui do
  system("cd #{ui["path"]} && #{MVN_CMD} clean")
end

desc "[Alias to :update] Update (git pull) all nakamur and ui projects."
task :up => :update do
end

desc "Update (git pull) all nakamura and ui projects."
task :update do
  g = Git.open(ui["path"])
  remote = ui["remote"] || "origin"
  branch = remote + "/" + (ui["branch"] || "master")
  puts "Updating #{ui["path"]}:#{branch}"
  puts g.pull(remote, branch)

  for p in nakamura do
    g = Git.open(p["path"])
    remote = p["remote"] || "origin"
    branch = remote + "/" + (p["branch"] || "master")
    puts "Updating #{p["path"]}:#{branch}"
    puts g.pull(remote, branch)
  end
end

desc "Rebuild the ui and nakamura projects."
task :rebuild do
  system("cd #{ui["path"]} && #{MVN_CMD} clean install")
  for p in nakamura do
    system("cd #{p["path"]} && #{MVN_CMD} clean install")
  end
end

desc "Rebuild just the app bundle to include any changed bundles without building everything."
task :fastrebuild do
  system("cd ../nakamura/app && #{MVN_CMD} clean install")
end

desc "Start a running server. Will kill the previously started server if still running."
task :run => [:kill] do
  app_file = nil
  Dir["../nakamura/app/target/org.sakaiproject.nakamura.app-*.jar"].each do |path|
    if !path.end_with? "-sources.jar" then
      app_file = path
    end
  end
  abort("Unable to find application version") if app_file.nil?

  CMD = "#{JAVA_CMD} -jar #{app_file} #{APP_OPTS}"
  p "Starting server with #{CMD}"

  pid = fork { exec( CMD ) }
  Process.detach(pid)
  File.open(".nakamura.pid", 'w') {|f| f.write(pid) }
end

desc "Kill the previously started server."
task :kill do
  pidfile = ".nakamura.pid"
  if File.exists?(pidfile)
    File.open(pidfile, "r") do |f|
      while (line = f.gets) do
        pid = line.to_i
        begin
          Process.kill("TERM", pid)
          puts "Killing pid #{pid}"
          while (sleep 5) do
            begin
              Process.getpgid(pid)
            rescue
              break
            end
          end
        rescue
          puts "Didn't find pid #{pid}"
        end
      end
    end
    rm pidfile
  end
end

# ==================
# = Set FSResource =
# ==================

desc "Set the fs resource configs to use the ui files on disc."
task :setfsresource => [:setuprequests] do
  # set fsresource paths
  # has to be a single URL POST, no post params (weird, I know)
  uiabspath = `cd #{ui["path"]} && pwd`.chomp
  ["/dev", "/devwidgets", "/tests"].each do |dir|
    url = "/system/console/configMgr/[Temporary%20PID%20replaced%20by%20real%20PID%20upon%20save]"
    url += "?propertylist=provider.roots,provider.file,provider.checkinterval"
    url += "&provider.roots=#{dir}"
    url += "&provider.file=#{uiabspath}#{dir}"
    url += "&provider.checkinterval=1000"
    url += "&apply=true"
    url += "&factoryPid=org.apache.sling.fsprovider.internal.FsResourceProvider"
    url += "&action=ajaxConfigManager"
    req = Net::HTTP::Post.new(url)
    req.basic_auth("admin", "admin")
    response = @localinstance.request(req)
    puts response.inspect
  end
end

# ===========================================
# = Creating users and groups =
# ===========================================

# Fix header setting for Net::HTTP
module Net::HTTPHeader
  def initialize_http_header(initheader)
      @header = { "Referer" => ["http://localhost:8080"] }
      return unless initheader
      initheader.each do |key, value|
        warn "net/http: warning: duplicated HTTP header: #{key}" if key?(key) and $VERBOSE
        @header[key.downcase] = [value.strip]
      end
  end
end

task :setuprequests do
  @uri = URI.parse("http://localhost:8080")
  @localinstance = Net::HTTP.new(@uri.host, @uri.port)
end

desc "Create #{num_users_groups} users."
task :createusers => [:setuprequests] do
  num_users_groups.times do |i|
    i = i+1
    puts "Creating User #{i}"
    req = Net::HTTP::Post.new("/system/userManager/user.create.html")
    req.set_form_data({
      ":name" => "user#{i}",
      "pwd" => "test",
      "pwdConfirm" => "test",
      "email" => "user#{i}@sakaiproject.invalid",
      ":sakai:pages-template" => "/var/templates/site/defaultuser",
      "firstName" => "User",
      "lastName" => "#{i}",
      "locale" => "en_US",
      "timezone" => "America/Los_Angeles",
      "_charset_" => "utf-8",
      ":sakai:profile-import" => "{'basic': {'access': 'everybody', 'elements': {'email': {'value': 'user#{i}@sakaiproject.invalid'}, 'firstName': {'value': 'User'}, 'lastName': {'value': '#{i}'}}}}"
    })
    req.basic_auth("admin", "admin")
    response = @localinstance.request(req)
    puts response
  end
end

desc "Make connections between each user and the next sequential user id."
task :makeconnections => [:setuprequests] do
  num_users_groups.times do |i|
    i = i+1
    nextuser = i % num_users_groups + 1

    puts "Requesting connection between User #{i} and User #{nextuser}"
    req = Net::HTTP::Post.new("/~user#{i}/contacts.invite.html")
    req.set_form_data({
      "fromRelationships" => "Classmate",
      "toRelationships" => "Classmate",
      "targetUserId" => "user#{nextuser}",
      "_charset_" => "utf-8"
    })
    req.basic_auth("user#{i}", "test")
    response = @localinstance.request(req)
    puts response

    puts "Accepting connection between User #{i} and User #{nextuser}"
    req = Net::HTTP::Post.new("/~user#{nextuser}/contacts.accept.html")
    req.set_form_data({
      "targetUserId" => "user#{i}",
      "_charset_" => "utf-8"
    })
    req.basic_auth("user#{nextuser}", "test")
    response = @localinstance.request(req)
  end
end

desc "Create #{num_users_groups} groups. Each is created by the user with the matching id."
task :creategroups => [:setuprequests] do
  num_users_groups.times do |i|
    i = i+1
    puts "Creating Group #{i}"
    req = Net::HTTP::Post.new("/system/userManager/group.create.html")
    req.set_form_data({
      ":name" => "group#{i}",
      ":sakai:pages-template" => "/var/templates/site/defaultgroup",
      ":sakai:manager" => "user#{i}",
      "sakai:group-title" => "Group #{i}",
      "sakai:group-description" => "Group #{i} description",
      "sakai:group-joinable" => "yes",
      "sakai:group-visible" => "public",
      "sakai:pages-visible" => "public",
      "_charset_" => "utf-8"
    })
    req.basic_auth("admin", "admin")
    response = @localinstance.request(req)
    puts response

    ["anonymous", "everyone"].each do |grant|
      req = Net::HTTP::Post.new("/~group#{i}.modifyAce.html")
      req.set_form_data({
        "principalId" => "#{grant}",
        "privilege@jcr:read" => "granted",
        "_charset_" => "utf-8"
      })
      req.basic_auth("admin", "admin")
      response = @localinstance.request(req)
      puts response
    end
  end
end

desc "Send messages between users."
task :sendmessages => [:setuprequests] do
  num_users_groups.times do |i|
    i += 1
    nextuser = i % num_users_groups + 1

    puts "Sending internal message: user#{i} => user#{nextuser}"
    puts send_internal_message "user#{i}", "user#{nextuser}", "test #{i} => #{nextuser}", "test body #{i} => #{nextuser}"

    puts "Sending smtp message: user#{i} => user#{nextuser}"
    puts send_smtp_message "user#{i}", "user#{nextuser}", "test #{i} => #{nextuser}", "test body #{i} => #{nextuser}"

    puts "Sending internal message: user#{nextuser} => user#{i}"
    puts send_internal_message "user#{nextuser}", "user#{i}", "test #{nextuser} => #{i}", "test body #{nextuser} => #{i}"

    puts "Sending smtp message: user#{nextuser} => user#{i}"
    puts send_smtp_message "user#{nextuser}", "user#{i}", "test #{nextuser} => #{i}", "test body #{nextuser} => #{i}"
  end
end

desc "Update and rebuild the ui and nakamura projects."
task :build => [:update, :rebuild]

desc "Create users, greate groups, make connections, send messages, set fs resources, clean the ui"
task :setup => [:createusers, :creategroups, :makeconnections, :sendmessages, :setfsresource, :cleanui]

desc "Clean, build and run"
task :default => [:clean, :build, :run]

