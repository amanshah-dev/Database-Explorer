create procedure update_synapseiiexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Synapse II', achievementCategoryId, userIds);
END;
$$;

alter procedure update_synapseiiexpert(bigint, bigint[]) owner to learner;
