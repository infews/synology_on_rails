# Add your own tasks in files placed in lib/tasks ending in .rake,
# for example lib/tasks/capistrano.rake, and they will automatically be available to Rake.

require_relative "config/application"
require "standard/rake"

Rails.application.load_tasks

# for colorization of output
require "rainbow/refinement"
using Rainbow

# for making calls for deployment
require "sshkit"
require "sshkit/dsl"
require "sshkit/sudo"

desc "Deploy to Synology NAS Docker"
task deploy: [:ensure_clean_git,
              :rebuild_assets,
              :build_image,
              :copy_image,
              :deploy_to_synology]

desc "Prevent cleaning & building with uncommitted changes"
task :ensure_clean_git do
  stdout, _stderr, _status = Open3.capture3("git status -s")

  unless stdout.empty?
    puts
    puts Rainbow("✋ Please commit all changes before deploying so app has latest & greatest\n").goldenrod.bold
    exit 1
  end
end

desc "Clobber and rebuild assets"
task :rebuild_assets do
  # clean and rebuild assets
  system "./bin/rails assets:clobber"
  system "./bin/rails assets:precompile"

  puts
  puts "🛄 " + Rainbow("Re-built assets in ").cyan.bold + Rainbow("./public \n").bold
  puts
end

desc "Build and save container image"
task :build_image do
  # Build container image
  puts
  puts "🛄🛄 " + Rainbow("Building container image for Intel").cyan.bold

  key = File.read("config/master.key")
  uid = 1032
  gid = 100
  build_cmd =
    "docker buildx build " +
      " --build-arg=\"MASTER_KEY=#{key}\"" +
      " --build-arg=\"UID\"=#{uid}" +
      " --build-arg=\"GID\"=#{gid}" +
      " --platform linux/amd64 -t dwfrank/meals ."

  system build_cmd

  puts
  puts "🛄🛄🛄 " + Rainbow("Saving image to ").cyan.bold + Rainbow("./tmp \n").bold

  # Get image to NAS
  system "docker save dwfrank/meals > tmp/dwfrank-meals.tar"
end

desc "Copy image to NAS"
task :copy_image do
  puts
  puts "🛄🛄🛄🛄 " + Rainbow("Copying image to NAS").cyan.bold

  system "cp tmp/dwfrank-meals.tar /Volumes/docker/meals"
  system "cp docker-compose.yml /Volumes/docker/meals"

  puts
  puts "🛄🛄🛄🛄 " + Rainbow("Done copying").cyan.bold
end

desc "Loads the built image into Docker on Filgate"
task :deploy_to_synology do
  include SSHKit::DSL

  puts
  puts "🛄🛄🛄🛄🛄 " + Rainbow("Deploying to NAS").cyan.bold

  nas_via_ssh = "balboa@192.168.10.63:822"

  pw = "kwijibo" # set your deploy password here; maybe fetch from 1Password using CLI
  send_password = SSHKit::MappingInteractionHandler.new("Password: " => pw)

  bin = "/usr/local/bin"
  docker_compose = "#{bin}/docker-compose"
  docker = "#{bin}/docker"

  on nas_via_ssh do
    # acutal mounted path for the docker folder
    within "/volume1/docker/meals" do
      sudo "#{docker_compose} down",
           interaction_handler: send_password
      sudo "#{docker} image rm --force dwfrank/meals",
           interaction_handler: send_password
      sudo "#{docker} load --input /volume1/docker/meals/dwfrank-meals.tar",
           interaction_handler: send_password
      sudo "#{docker_compose} create",
           interaction_handler: send_password
    end
  end

  puts
  puts "🛄🛄🛄🛄🛄 " + Rainbow("Current container stopped").cyan.bold
  puts "🛄🛄🛄🛄🛄🛄 " + Rainbow("Old images removed").cyan.bold
  puts "🛄🛄🛄🛄🛄🛄🛄 " + Rainbow("New image loaded").cyan.bold
  puts "🛄🛄🛄🛄🛄🛄🛄🛄 " + Rainbow("New container created").cyan.bold
  puts "🛄🛄🛄🛄🛄🛄🛄🛄🛫 " + Rainbow("Time to map service to container in NAS Web Station").cyan.bold
  puts
end
