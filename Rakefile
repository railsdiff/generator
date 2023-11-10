# frozen_string_literal: true

require 'json'

VERBOSE = ENV.key?('VERBOSE')

task 'default' => 'update_rails_repo'

directory 'tmp/rails'

file 'tmp/rails/rails' => 'tmp/rails' do |t|
  puts 'Cloning Rails repo' if VERBOSE
  sh("git clone https://github.com/rails/rails.git #{t.name} > /dev/null 2>&1", verbose: VERBOSE)
end

task 'update_rails_repo' => 'tmp/rails/rails' do |t|
  puts 'Updating Rails repo' if VERBOSE

  cd(t.prerequisites.first, verbose: VERBOSE) do
    sh('git fetch origin --tags > /dev/null', verbose: VERBOSE)
  end
end

file 'tmp/rails/generator' => 'tmp/rails' do |t|
  generator = <<~GEN
    activesupport_path = File.expand_path('../activesupport/lib', __FILE__)
    railties_path = File.expand_path('../railties/lib', __FILE__)
    $:.unshift(activesupport_path)
    $:.unshift(railties_path)
    require "rails/cli"
  GEN
  File.write(t.name, generator)
end

rule(%r{tmp/rails/.*} => ['tmp/rails/rails', 'tmp/rails/generator']) do |t|
  cd(t.source, verbose: VERBOSE) do
    tag = t.name.pathmap('%f')
    sh("git reset --hard #{tag} > /dev/null 2>&1", verbose: VERBOSE)
  end
  cp_r(t.source, t.name, verbose: VERBOSE)
  cp('tmp/rails/generator', t.name, verbose: VERBOSE)
end

directory 'generated'

rule(%r{generated/.*} => ['generated']) do |t|
  source = t.name.pathmap('%{generated,tmp/rails}p')
  Rake::Task[source].invoke # only invoke this task if we need generated app

  puts format('Generating: %<name>s', name: t.name) if VERBOSE

  rm_rf(t.name, verbose: VERBOSE) if Dir.exist?(t.name)
  sh(generator_command(source, t.name), verbose: VERBOSE)
  sed_commands(t.name).each do |expression, file_path|
    sh(%(sed -E -i '' "#{expression}" #{file_path}), verbose: VERBOSE)
  end
  `test -d "#{t.name}/railsdiff/.git" && rm -rf "#{t.name}/railsdiff/.git"`
  sh("mv #{t.name}/railsdiff/* #{t.name}/.", verbose: VERBOSE)

  hidden_glob = "#{t.name}/railsdiff/.??*"
  `test -n "$(ls #{hidden_glob})" && mv #{hidden_glob} #{t.name}/.`

  rm_rf(source, verbose: VERBOSE)
end

def generator_command(source_path, dest_path)
  if source_path.split(%r{/}).last =~ /v2.3/
    "ruby #{source_path}/railties/bin/rails #{dest_path}/railsdiff > /dev/null"
  else
    "ruby #{source_path}/generator new #{dest_path}/railsdiff --skip-bundle --skip-webpack-install > /dev/null"
  end
end

def sed_commands(base_path) # rubocop:disable Metrics/MethodLength
  [
    [
      %{s/^(ARG RUBY_VERSION=).*$/\\1your-ruby-version/},
      "#{base_path}/railsdiff/Dockerfile"
    ],
    [
      %{s/^ruby ('|\\").*['\\"]$/ruby \\1your-ruby-version\\1/},
      "#{base_path}/railsdiff/Gemfile"
    ],
    [
      's/.*/your-encrypted-credentials/',
      "#{base_path}/railsdiff/config/credentials.yml.enc"
    ],
    [
      "s/'.*'/'your-secret-token'/",
      "#{base_path}/railsdiff/config/initializers/cookie_verification_secret.rb"
    ],
    [
      "s/'.*'/'your-secret-token'/",
      "#{base_path}/railsdiff/config/initializers/secret_token.rb"
    ],
    [
      "s/'.{20,}'/'your-secret-token'/",
      "#{base_path}/railsdiff/config/initializers/session_store.rb"
    ],
    [
      's/(secret_key_base:[[:space:]])[^<].*$/\1your-secret-token/',
      "#{base_path}/railsdiff/config/secrets.yml"
    ]
  ].select { |_expression, file_path| File.exist?(file_path) }
end
