create function url_encode(data bytea) returns text
    immutable
    language sql
as
$$
    SELECT translate(encode(data, 'base64'), E'+/=\n', '-_');
$$;

alter function url_encode(bytea) owner to learner;
