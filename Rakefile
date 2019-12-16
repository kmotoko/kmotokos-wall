require 'html-proofer'

task :test do
    sh "bundle exec jekyll doctor"
    options = {
        :assume_extension => true,
        :check_favicon => true,
        :check_opengraph => true,
        :check_html => true,
        :check_img_http => true,
        :check_sri => true,
        :enforce_https => true,
        :internal_domains => [
            "kmotoko.com",
            "www.kmotoko.com"
        ],
         # 999=Request Denied Error, might happen if the server thinks you are a bot
        :http_status_ignore => [999],
        :validation => {
            :report_missing_doctype => true,
            :report_script_embeds => true,
            :report_missing_names => true,
            :report_invalid_tags => false
        },
        :typhoeus => {
            :headers => {
                "User-Agent" => "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36",
                "Accept-Encoding" => "gzip"
            },
            :followlocation => true,
            :connecttimeout => 10,
            :timeout => 30
        }
    }
    sh "JEKYLL_ENV=development bundle exec jekyll build --strict_front_matter"
    HTMLProofer.check_directory("./_site", options).run
    sh "JEKYLL_ENV=production bundle exec jekyll build --strict_front_matter"
    HTMLProofer.check_directory("./_site", options).run
end
